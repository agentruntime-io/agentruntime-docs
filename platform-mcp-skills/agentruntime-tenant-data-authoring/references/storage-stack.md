# Platform storage stack (for external agents)

When designing CRM, inbox, or extraction flows, pick the right **durable layer**:

| Layer | What it stores | Agent/workflow access | Author with |
|-------|----------------|----------------------|-------------|
| **Tenant Data** | Structured rows (people, accounts, history events) | `tenantdata_*` MCP on provisioned instance | This skill + `logical-schema.schema.json` |
| **Work files** | Blobs (PDFs, exports, large paste attachments) | **`files_upload_content`**, **`files_list`**, **`files_get`**, **`files_mcp_provision`** | Store `file_id` in Tenant Data columns |
| **Surfaces (control-service)** | App shell, views, queries (JSONB) | Render/action API; `database.query` binds | [agentruntime-surfaces-authoring](../agentruntime-surfaces-authoring/SKILL.md) |
| **Vault** | Secrets, MCP inline config | Connections / instance wiring | Platform MCP vault tools (risk subgroup) |
| **Workflow run context** | Ephemeral step outputs | `{{steps.*}}` templates | Workflow skill |

## Practical split (CRM + paste extract)

1. **`extraction_events`** — append-only: raw text, run id, source, `created_at` (Tenant Data).
2. **`people` / `accounts`** — upsert on stable keys (`email`, `domain`, `external_id`).
3. **`people_field_history`** — append-only field snapshots linked to `extraction_event_id`.
4. Optional **Work file** for original upload; Tenant Data row holds `source_file_id`.

Do not store large unstructured blobs in many `string` columns when a file reference suffices.

## Schema versioning

Tenant Data keeps **`tenant_schema_versions`** (control DB) + **`current_schema_version`** on each database. Agents should read **`api_contract`** after every migrate, not cache column lists from memory.

See monorepo: `docs/surfaces/SURFACES_STORAGE.md` for Surfaces-specific tables (`surface_apps`, `surface_app_views`, …).
