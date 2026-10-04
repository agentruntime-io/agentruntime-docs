# Platform MCP — Tenant Data apply loop

Call **`ar_get_started`** first. Use **`tenant_id`** from response when tool inputs omit it.

## Tools

| Tool | BFF route | Purpose |
|------|-----------|---------|
| `tenantdata_logical_schema` | (embedded) | JSON Schema for `{ "tables": [] }` |
| `tenantdata_databases_list` | GET `…/databases` | List tenant databases |
| `tenantdata_databases_create` | POST `…/databases` | Create DB + initial schema |
| `tenantdata_schema_migrate_preview` | POST `…/schema/migrate/preview` | Preflight SQL + blockers |
| `tenantdata_schema_migrate` | POST `…/schema/migrate` | Apply migration ops |
| `tenantdata_api_contract_get` | GET `…/api-contract` | Published columns + upsert policy |
| `tenantdata_mcp_provision` | POST `…/mcp/provision` | Create/update MCP instance |

Work files (blobs): `files_upload_content`, `files_list`, `files_get`, `files_mcp_provision` — see `storage-stack.md`.

**Validate workflows:** pass `tenantdata_schema` (same `{ tables }` as api-contract) to **`workflows_validate`** when not using Studio auto-resolve.

## Create body (API)

```json
{
  "database_id": "crm",
  "database_name": "CRM",
  "schema": { "tables": [ … ] }
}
```

`database_id` is the logical **database_key** (stable id). Display name is human-readable.

## Migrate operations (incremental)

Prefer **`add_table`** / **`add_column`** for v2 changes. Ops match Console migrate API (`add_table`, `add_column`, `drop_column`, `update_table`, …). Preview first when agents automate.

## Workflow binding

After provision, set workflow MCP steps:

- `instance_id`: returned `mcp_instance_id`
- `tool_name`: `tenantdata_insert` etc.
- `tool_args.table`: logical table **name** (not display name)

Validate with **`workflows_validate`** (BFF auto-resolves `tenantdata_schema` when instance is bound).
