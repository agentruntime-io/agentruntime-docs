# Validation and handoff

This portable bundle does **not** ship offline lint scripts. Use **server dry-run** as the primary validation path when a deployment is available. See [dry-run.md](dry-run.md) for checks, request fields, and issue codes.

## Validation stages

| Stage | Evidence | Does not establish |
|-------|----------|-------------------|
| Bundled JSON Schema | Authoring-profile shape | Lua compile, dependencies, tenant tool schema |
| **Server dry-run** | Graph, deps, templates, Lua syntax/execute, MCP bindings/resolve, input schema, tenantdata upsert trace, optional mapper fixtures | Live tool data, SQL, every branch |
| Controlled run | Behavior for tested inputs/account | All future responses, races, recovery |

Use a Draft 2020-12 validator on the bundled schema separately when available.

## API

`POST /v1/workflows/{workflow_id}/validate`

Minimum body:

```json
{
  "graph": { "steps": [] },
  "params": { "trigger_payload": {} }
}
```

For Tenant Data workflows, include published logical schema or use Studio (auto-resolves from bound databases):

```json
{
  "graph": { "steps": [] },
  "params": { "trigger_payload": {} },
  "tenantdata_schema": {
    "tables": [
      {
        "name": "my_table",
        "on_duplicate": "upsert",
        "upsert_columns": ["id", "name"],
        "columns": [
          { "name": "id", "type": "string", "required": true },
          { "name": "name", "type": "string" }
        ]
      }
    ]
  }
}
```

Optional: `mapper_fixtures` (array of cases with `step_id`, `scope`, `expect_row` / `expect_rows`), `mock_lua_scopes`, `mock_step_outputs`, `checks`, `skip_checks`, `step_ids`, `assume_placeholder_outputs`.

`input.*` resolves after `ApplyInputSchemaDefaults` materializes top-level keys from `input_schema`. Pipe modifiers (`| omit`, literals, coalesce): [template-modifiers.md](template-modifiers.md).

Inspect `checks_performed` and `issues` in the response. Errors fail validation; warnings require review.

## Default checks (summary)

`graph`, `dependencies`, `step_config`, `templates`, `lua_syntax`, `mcp_bindings`, `disabled_steps`, `input_schema`, `lua_fixture`, `tenantdata_upsert`, plus `mcp_resolve` when applicable. Add `mapper_fixtures` by including fixtures in the request body.

## Studio workflow

1. Create Tenant Data database and publish schema; provision Tenant Data MCP instance.
2. Import workflow JSON; map `REPLACE_*` MCP instances and `REPLACE_CHILD_*` child workflows at **Review import**.
3. Import/publish child workflows before orchestrators.
4. **Dry-run** — Studio sends graph + start params + auto-resolved `tenantdata_schema`.
5. Save → Publish → re-read published graph (ids, scripts, bindings, `input_schema`).
6. Controlled run with real principal / connections.

## Mapper fixtures (API)

Pass inline `mapper_fixtures` with `tenantdata_schema` to execute mapper Lua against mock `body`/`item`/`run` and validate row keys/types. Each case needs a distinct `step_id` (runtime looks up by id, not parent path).

Repository maintainers may also run the in-repo CI corpus via `.cursor/skills/.../validate-tenantdata-fixtures.mjs` — that path is not part of this portable bundle.

## Platform MCP validate tool

When connected to Platform MCP, prefer **`workflows_validate`** with inline `graph` (and `params` / `tenantdata_schema` / `mapper_fixtures` / `mock_lua_scopes` / `step_ids` when needed) instead of calling `POST /v1/workflows/{id}/validate` directly. Issue objects include `json_path`, `suggestion`, `schema_ref`, and `fixable` when the server catalog has hints. Same checks as [dry-run.md](dry-run.md).

## Remaining gaps

- **`tenantdata_logical_schema`** (Platform MCP) returns the logical `{ tables[] }` JSON Schema; **`tenantdata_databases_create`** / **`tenantdata_api_contract_get`** apply and read published shape. Otherwise use Studio export or inline `tenantdata_schema` on validate.
- Run-start `input_schema` may remain env-gated (`WORKFLOW_INPUT_SCHEMA_VALIDATION_ENABLED`).
- `connection.primary_instance_id` may false-alarm in template dry-run until whitelist updated.
- Studio UUID slug normalization — verify ids after publish.
- Heuristic `tenantdata_upsert` cannot prove every branch; untraced `rows: ["{{item}}"]` may warn.

## Changelog

- **2026-09-11 / server-first validation:** removed bundled Node lint scripts; documented `tenantdata_upsert`, `lua_fixture` (composite bodies), `mapper_fixtures`, and Studio schema auto-resolve. Added [dry-run.md](dry-run.md).
- **2026-09-11 / template pipe modifiers:** `| omit`, defaults, coalesce — [template-modifiers.md](template-modifiers.md).
- **2026-09-10 / reconciliation:** primary-instance identity, Studio import round-trip, strict Composio unwrap guidance in [connectors-and-data.md](connectors-and-data.md).
