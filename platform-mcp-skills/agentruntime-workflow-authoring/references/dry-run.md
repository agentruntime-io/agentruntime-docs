# Dry-run vs runtime

> **Dry-run = compile and statically validate the workflow. Run = execute the workflow.**

Dry-run answers whether the graph is structurally valid and whether declared Tenant Data upserts are wired to Lua row builders. It does **not** prove live MCP responses, SQL success, or every branch of business logic.

Responses include **`readiness`** (`ready` | `blocked`) and **`blockers[]`** in addition to **`valid`**. A graph can be `valid: true` but `readiness: blocked` when readiness-blocking warnings apply (for example missing MCP input schema). Use **`check_results`** for per-check `passed` | `failed` | `blocked` | `not_applicable`.

**`validation_evidence`** in the validate response carries **`graph_hash`**, **`workflow_graph_schema_hash`**, **`validated_at`**, **`readiness`**, and **`checks_performed`**. After each validate call, the BFF stores the full validate payload in control-service **`workflow_graph_dry_runs`** keyed by `(workflow_id, graph_hash)` — independent of publish. Fetch the last result with `GET /v1/workflows/{id}/graph-dry-runs?graph_hash=…`.

**Publish** does **not** run dry-run and does **not** block on `valid` or `readiness`. Publishing saves a version; dry-run is advisory for authoring and ops.

## API

`POST /v1/workflows/{workflow_id}/validate` on the deployment BFF (AgentRuntime).

```json
{
  "graph": { "steps": [], "input_schema": {} },
  "params": { "trigger_payload": { "topic": "actual value" } },
  "tenantdata_schema": { "tables": [] },
  "mapper_fixtures": [],
  "mock_lua_scopes": {},
  "mock_step_outputs": {},
  "assume_placeholder_outputs": false,
  "checks": [],
  "skip_checks": [],
  "step_ids": []
}
```

Only `graph` is required. Prefer the **default** check set (omit `checks`) for whole-workflow validation. Custom `checks` **replace** the default set — inspect `checks_performed` in the response.

### Request fields

| Field | Purpose |
|-------|---------|
| `params` / `payload` | Start input for `input.*` template resolution and `input_schema` validation |
| `tenantdata_schema` | Optional. Logical schema for `tenantdata_upsert` and `mapper_fixtures`. **Omit when authenticated** — BFF resolves from Tenant Data MCP bindings + `api-contract`. Pass explicitly for unsigned/external validate or to override. |
| `mapper_fixtures` | Inline fixture cases: mock `body`/`item`/`run`, step id, expected row fields |
| `mock_lua_scopes` | Per-step `MapperFixtureScope` for composite/top-level `lua_fixture` execution |
| `mock_step_outputs` | Synthetic completed step outputs (does not execute Lua by itself) |
| `step_ids` | Limit step-scoped checks to specific steps |

**Authenticated callers (Studio, PAT, API key):** when the graph uses `tenantdata_*` MCP tools, BFF auto-fetches published `api-contract` for bound Tenant Data databases and supplies `tenantdata_schema` (read-only). Unsigned validate without tenant context skips this and may warn `tenantdata_schema_unavailable`.

## Default server checks

| Check ID | What it validates |
|----------|-------------------|
| `graph` | Steps exist, supported types, studio nodes |
| `graph_schema` | Canonical JSON Schema on publishable graph fields (`steps`, `input_schema`, …); Studio-only keys like `nodes` are ignored |
| `dependencies` | `depends_on` targets exist, no cycles; every `{{steps.<id>.*}}` ref reachable via `depends_on` (and `input_allowlist` when set) |
| `step_config` | Required fields per step type |
| `templates` | `{{…}}` syntax; **`input.*` must resolve** when start params provided; MCP `tool_args` vs published or Studio `inputSchema` |
| `lua_syntax` | Script compiles |
| `mcp_bindings` | `instance_id` or `server_url` present |
| `mcp_resolve` | Auto-enabled when tenant + MCP steps + resolver wired |
| `disabled_steps` | Warn on `enabled=false` |
| `input_schema` | Materialized defaults + provided params vs graph schema |
| `lua_fixture` | Execute Lua in top-level and composite bodies + `while_eval_script`; optional `mock_lua_scopes` |
| `tenantdata_upsert` | Trace `tenantdata_insert` `rows[]` → Lua vs `upsert_columns` when schema supplied; warn if inserts exist but schema missing |

`mapper_fixtures` runs only when the request includes a nonempty `mapper_fixtures` array (not in the default set).

## Issue codes (selected)

| Code | Severity | Meaning |
|------|----------|---------|
| `template_unresolved` | error | Required `input.*` (or template) did not resolve |
| `template_missing_dependency` | error | `{{steps.*}}` ref not reachable via `depends_on` |
| `for_each_items_unresolved` | error | `for_each_items` stayed an unresolved `{{…}}` string |
| `for_each_items_invalid` | error | List coercion failed (may also fire when upstream `steps.*` output absent) |
| `mcp_connection_resolution_conflict` | error | Same instance with conflicting principal/workspace modes |
| `mcp_connection_resolution_principal` | warning | Composio + `principal` — needs principal at run |
| `connection_root_multi_instance` | warning | Root `connection.*` with 2+ MCP instances |
| `input_resolver_no_model` | warning | Resolver without `model` / `model_ref` |
| `graph_schema_validation` | error | Graph JSON fails canonical schema (unknown/typo fields on steps) |
| `mcp_tool_input_schema_unavailable` | warning (blocks readiness) | Bound MCP step without loadable input schema |
| `mcp_tool_arg_deferred` | warning | Tool arg still contains `{{…}}` — validated at dispatch |
| `tenantdata_schema_unavailable` | warning (blocks readiness) | `tenantdata_insert` without schema |
| `tenantdata_upsert_column_missing` | error | Upsert column not assigned in traced Lua mapper |
| `tenantdata_rows_untraced` | warning | Could not trace `rows[]` to a Lua row builder |
| `lua_fixture_return_type` | error | Lua did not return a table |
| `lua_fixture_runtime` | warning | Lua execution failed (often missing mock data) |
| `mapper_fixture_failed` | error | Inline mapper fixture did not match expected row |

Full template rules: [template-modifiers.md](template-modifiers.md). Connection identity: [connectors-and-data.md](connectors-and-data.md).

## Template resolution (summary)

- **`input.*`** — must resolve at dry-run when params/schema defaults provided (pipe modifiers: `| omit`, `| "default"`, coalesce chains).
- **`steps.*`** — structural only (step exists + dependency reachable); upstream output not required.
- **`run.*`, `connection.*`, loop scope** — compile-only whitelist; no live run context simulated.

## What dry-run still does not prove

| Gap | Notes |
|-----|--------|
| Live MCP / SQL | Use a controlled run |
| Every Lua branch / nil on all paths | Use `mapper_fixtures` + `mock_lua_scopes` or a run |
| `connection.primary_instance_id` templates | Compile whitelist may not include this key yet — see [runtime-contract.md](runtime-contract.md) |
| Run-start `input_schema` enforcement | May be env-gated separately from dry-run |
| Child `workflow_call` graph validity | Validate each published child workflow separately |

## Pass / fail

| Result | Meaning |
|--------|---------|
| Dry-run **pass** | Graph is well-formed for static checks; does not guarantee run success |
| Dry-run **fail** | Fix authoring before relying on a run |
| **`readiness: blocked`** | Advisory — review blockers; publish is still allowed |
| Warnings | Review — may be missing schema, untraced rows, or data-dependent Lua |

Do not treat dry-run pass as proof of correct Composio envelopes, pagination completion, or cursor safety under partial failure.
