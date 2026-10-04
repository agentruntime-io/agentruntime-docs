# History and upsert patterns

Tenant Data has **no automatic row audit trail**. Model history explicitly.

## Current state (upsert)

```json
{
  "name": "people",
  "display_name": "People",
  "on_duplicate": "upsert",
  "conflict_columns": ["email"],
  "upsert_columns": ["full_name", "title", "account_id", "updated_at"],
  "columns": [
    { "name": "id", "type": "uuid", "required": true, "primary_key": true, "auto_generate": true },
    { "name": "email", "type": "email", "required": true, "unique": true },
    { "name": "full_name", "type": "string" },
    { "name": "title", "type": "short_text" },
    { "name": "account_id", "type": "uuid", "references": { "table": "accounts", "column": "id" } },
    { "name": "updated_at", "type": "datetime" }
  ]
}
```

Workflow **`tenantdata_insert`** must include **every** `upsert_columns` key as a non-nil value (Lua drops nil keys). See workflow skill `tenantdata-mcp.md`.

## Append-only history

```json
{
  "name": "extraction_events",
  "display_name": "Extraction events",
  "on_duplicate": "fail",
  "columns": [
    { "name": "id", "type": "uuid", "required": true, "primary_key": true, "auto_generate": true },
    { "name": "run_id", "type": "uuid", "required": true },
    { "name": "raw_text", "type": "string" },
    { "name": "source_file_id", "type": "string" },
    { "name": "created_at", "type": "datetime", "required": true }
  ]
}
```

Per extracted field change, insert into **`people_field_history`** (fail on duplicate) with `person_id`, `field_name`, `old_value`, `new_value`, `event_id`.

## Idempotency (client UUIDs)

| Policy | Typical tables | Retry behavior |
|--------|----------------|----------------|
| `fail` | History, change_events | New uuid per event |
| `ignore` | `sources` keyed by capture/request id | Same id → second insert skipped |
| `upsert` | Current entities | Must supply all effective `upsert_columns` |

**Pitfall:** `ignore` on **`review_items`** with a stable id hides updated proposals when a new ingestion run re-extracts — use a new review id or a dedicated apply/replace workflow. Platform MCP: **`tenantdata_idempotency_patterns`**.

## Accounts

Mirror `people` with `on_duplicate: "upsert"` on `domain` or `external_id`. Link tables optional if FK columns on `people` are enough for v1.
