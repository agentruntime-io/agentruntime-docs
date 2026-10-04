---
name: agentruntime-tenant-data-authoring
description: Author Tenant Data logical schemas ({ tables[] }), history/CRM patterns, and apply them via Platform MCP or Console Import JSON. Use with workflow authoring for tenantdata_* MCP steps.
---

Design **durable row storage** (PostgreSQL per-tenant schemas) that agents and workflows access through **Tenant Data MCP** (`tenantdata_insert`, `tenantdata_select`, …).

## Hosted copy (external agents)

**Browse:** https://github.com/agentruntime-io/agentruntime-docs/tree/main/platform-mcp-skills/agentruntime-tenant-data-authoring

**Fetch SKILL.md:** https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/agentruntime-tenant-data-authoring/SKILL.md

**Logical schema JSON Schema:** call Platform MCP **`tenantdata_logical_schema`** or fetch [schemas/logical-schema.schema.json](schemas/logical-schema.schema.json) (mirrors `tools/tenantdata-logical.schema.json` in platform-mcp for embed).

Platform MCP: **`ar_get_started`** → `documentation.tenant_data_authoring_skill_*` and playbook **`design_solution`**.

## Read first

| File | When |
|------|------|
| [references/storage-stack.md](references/storage-stack.md) | Where data lives (Tenant Data vs files vs Surfaces JSONB) |
| [references/history-and-upsert.md](references/history-and-upsert.md) | CRM current state + append-only history tables |
| [references/platform-mcp-tenant-data.md](references/platform-mcp-tenant-data.md) | Apply schema, provision MCP, read api-contract |
| [schemas/logical-schema.schema.json](schemas/logical-schema.schema.json) | Validate `{ "tables": [ … ] }` before apply |
| [agentruntime-workflow-authoring](../agentruntime-workflow-authoring/SKILL.md) | `tenantdata_*` steps, upsert/Lua rules (`tenantdata-mcp.md`) |

## Authoring rules (summary)

1. Body shape: `{ "tables": [ { "name", "display_name?", "on_duplicate?", "conflict_columns?", "upsert_columns?", "columns": [ … ] } ] }`.
2. Every table needs a primary key (column with `"primary_key": true` or explicit PK columns). Prefer `uuid` + `"auto_generate": true` for entity ids.
3. **`on_duplicate: "upsert"`** for current-state entities; use **`fail`** or **`ignore`** for append-only history/event tables.
4. Logical **`references`** are documentation today (Postgres FK enforcement is limited); still model FK columns for agent clarity.
5. After apply: **Provision MCP** on that database; workflow `instance_id` must target **that** instance (see workflow skill).
6. Do not invent MCP tool argument keys — use `catalog_tools_schema` for `tenantdata_*`.

## Apply (agents)

| Step | Platform MCP |
|------|----------------|
| Validate shape | `tenantdata_logical_schema` (meta) + JSON Schema client-side |
| Create DB + v1 schema | `tenantdata_databases_create` |
| Incremental change | `tenantdata_schema_migrate_preview` → `tenantdata_schema_migrate` |
| Read published contract | `tenantdata_api_contract_get` |
| Wire workflows | `tenantdata_mcp_provision` → copy `mcp_instance_id` into workflow `mcp_call.instance_id` |

Requires PAT with tenant-data access (same wires as Console Databases).

## Examples in monorepo

`workflow-examples/packages/*/tenant-data/*.schema.json` — copy patterns, not runtime IDs.
