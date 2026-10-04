# Phased CRM-style delivery (external agents)

Do not ship a single “complete CRM” in one pass. Deliver **evidence → review → apply → UI** in order.

## Phase v1 — Ingest to review (no entity writes)

- Tenant Data: `sources`, `ingestion_runs`, `review_items` with **`on_duplicate: ignore`** on evidence tables where client ids are stable.
- Workflow: preserve text → `llm_call` → store **raw** `result` (or `result.text`) in `proposed_action` JSON → queue `review_items.status = pending`.
- **Do not** insert into `people` / `claims` until apply exists.

**Idempotency:** Reusing the same `capture_id` skips duplicate `sources` inserts. A new run id can create another `ingestion_runs` row while `review_items` with the same id is also skipped — use a **new review row id** per re-extraction or an apply workflow. See **`tenantdata_idempotency_patterns`**.

## Phase v2 — Parse extraction

- Add `lua_script` after LLM: **`json.decode(steps[<llm>].result.text)`** (see `llm-step-results.md`).
- Validate shape before writing `claims` rows (still proposals until apply).

## Phase v3 — Apply (platform gap if no single transaction tool)

- Optimistic concurrency: `expected_version` on current entities.
- Atomically update current row + `entity_versions` + `change_events` (or document that separate MCP inserts are **not** transactional).
- Do not claim “CRM live” until this phase runs in production.

## Phase v4 — Surfaces app (optional)

- **`surfaces_schema_catalog`** first (not monorepo raw fetch).
- `surfaces_apps_*` + `database.query` bindings to Tenant Data.
- **`surfaces_apps_put_gate_bindings`** when workflows use `human_task`.

## Evidence bar (each phase)

| Phase | Required evidence |
|-------|-------------------|
| v1 | `tenantdata_api_contract_get`, `mcp_instance_id`, `workflows_validate` valid + `tenantdata_schema` |
| v2 | Sample run with parsed JSON in review payload |
| v3 | Apply workflow run id + version checks |
| v4 | Published app + optional gate bindings |
