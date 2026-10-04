# App Studio wiring (MCP agents)

**Problem:** Saving view JSON and query JSON is **not** the same as a **working app**. App Studio shows “attached workflows” and browse **data sources** from what you persist and what **compile** accepts.

There is **no** Platform MCP tool named “attach workflow” or “attach data source”. Wiring is **declarative in JSON** plus a few dedicated PUT routes.

## What “attachment” means in AgentRuntime

| Studio concept | What actually persists | MCP / API |
|----------------|------------------------|-----------|
| **Attached workflows** (view header) | Top-level `workflows[]` on the **view document** **and** every `workflow_ref` inside `bind` / `events.*.action` | `surfaces_apps_put_view` — compile upserts `surface_app_workflow_usage` |
| **Run / approve actions** | Button `events.onClick.action` with `type`: `workflow.run`, `workflow.operation`, or `workflow.cancel` | Same PUT view body |
| **Data source (`database.query`)** | Row in `surface_app_queries` + `bind.source: "database.query"` with matching `query_id` | `surfaces_apps_put_query` **before** view binds reference the id |
| **HITL gate → view route** | `surface_app_gate_bindings` (separate from view tree) | `surfaces_apps_put_gate_bindings` |

Console parity: monorepo `docs/surfaces/CONSOLE_GUIDE.md` (Preview tab starts runs; Queries tab defines SQL).

## Order of operations (agents)

```text
1. Publish workflows → real workflow UUIDs (workflows_publish_version).
2. surfaces_apps_create → app_id.
3. surfaces_apps_put_query for each query_id used in tables/lists.
4. surfaces_apps_put_view for each view_id in navigation.
   - Include view.workflows[] for Studio’s attached list.
   - Include actions, not only display binds.
5. surfaces_apps_put_gate_bindings when human_task + dedicated review view.
6. Verify (below) before claiming “app works”.
```

## Capture / ingest views (common CRM mistake)

**Insufficient:** Text/Label components bound to `workflow.node.output` or static copy that *describes* the ingestion workflow. That does **not** start a run.

**Required for operator launch:**

1. View `workflows[]` includes the **ingestion** workflow UUID (`role: "primary"` is typical).
2. At least one **Button** (or Preview-equivalent flow) with:

```json
"events": {
  "onClick": {
    "action": {
      "type": "workflow.run",
      "workflow_ref": "<published-ingestion-workflow-uuid>",
      "params": { }
    }
  }
}
```

3. Optional **correction / review** workflow: separate UUID + `workflow.operation` buttons on `human_task` step ids, **or** gate bindings pointing review tasks at a review view.

Reference shape: monorepo `docs/surfaces/examples/tiktok/publish-view.json` (gate approve/reject) + `SURFACES_SCHEMA.md` §6 Actions.

## Browse / list views (`database.query`)

**Insufficient:** `surfaces_apps_put_view` with `database.query` bind but **no** query document, or SQL that does not match Tenant Data tables.

**Required:**

1. `surfaces_apps_put_query` with same `query_id` as in the bind (`pending_posts`, `people_list`, etc.).
2. SQL is **read-only SELECT** scoped to tables that exist (Tenant Data skill + provisioned MCP).
3. View `workflows[]` may use `role: "data"` when the view is browse-only (see `queue-view.json` example).

`database.query` binds do **not** use `workflow_ref`; compile focuses on workflow bindings. A broken browse view usually means **missing query row**, bad SQL at render, or wrong `query_id` — not `import_pending`.

## Compile status (`bind_compile`)

Every `surfaces_apps_put_view` runs compile and stores `bind_compile` on the view row.

| `bind_compile.status` | Meaning |
|-----------------------|---------|
| `ok` | Workflow refs are UUIDs; bound `node_id` values exist on **active published** graph |
| `import_pending` | Non-UUID `workflow_ref` — resolve in Studio or replace with published UUID |
| `error` | Unknown workflow, missing active version, or `node_id` not in graph |

**Agents must read back:**

- `surfaces_apps_get_view` → `bind_compile`
- `surfaces_apps_list` / `surfaces_apps_get` → aggregate `bind_compile_status` on the app

Do **not** treat “view saved” as “bindings valid” without `bind_compile.status === "ok"`.

Gate bindings: `surfaces_apps_list_gate_bindings` after save if you use `workflow.operation` approve/reject.

## Final verification checklist

Before claiming the app is usable:

| Check | How |
|-------|-----|
| Workflows published | UUIDs in JSON match `workflows_list` / Studio published ids |
| Compile clean | Every view: `bind_compile.status` is `ok` (not `import_pending` / `error`) |
| Queries exist | `surfaces_apps_get_query` for each `query_id` in browse binds |
| Can **start** primary flow | View has `workflow.run` action **or** documented operator path via Studio Preview |
| Can **approve** review | `workflow.operation` on correct `node_id` + optional `surfaces_apps_put_gate_bindings` |
| Can **read rows** | `surfaces_render` with `mode: "browse"` (and params your query expects) returns table data — not an empty error banner |
| Run mode (optional) | `surfaces_render` with `mode: "run"` + `run_id` after `workflow_run` shows live step bindings |

Use **`surfaces_render`** for evidence in agent reviews; list `checks_performed` with app id, view ids, and compile statuses.

## Product gaps (do not pretend otherwise)

- No MCP “attach workflow to app” besides editing view JSON + compile side effects.
- No single MCP “compile only” — compile runs on view PUT.
- Browse data requires query PUT + Tenant Data tables; MCP does not auto-wire MCP instance to query — SQL and provisioning must match tenant-data skill.

When Studio and MCP disagree, monorepo **`docs/surfaces/SURFACES_RUNTIME_CONTRACTS.md`** is authoritative for compile and render modes.
