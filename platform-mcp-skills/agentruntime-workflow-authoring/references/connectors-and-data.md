# MCP, identity, and Tenant Data

## Discovery before authoring

**Platform MCP (preferred):** See [platform-mcp-discovery.md](platform-mcp-discovery.md). Use `catalog_tools_search` for `tool_name` + `instance_id`, then `catalog_tools_schema` for `input_schema` / `output_schema`. Instances are per connector (shared across tools on that connector), not per tool.

For each `mcp_call` step you still need: exact `tool_name`, schemas, tenant `instance_id`, and whether the instance is connected (`has_connection`). First-party and Composio implementations of the same integration may describe the same operation but do not necessarily share names, arguments, envelopes, pagination, scopes, or errors. Do not translate between them by name alone.

Prefer a resolved tenant `instance_id`. A `server_url` or `server_id` is not a credential. URL-only resolution/instance creation can depend on deployment settings; do not promise automatic placeholder remapping. Multiple Composio toolkits may share a URL. Tenant Data instances may select different logical databases at the same URL.

## Identity

For embedded per-customer OAuth set `connection_resolution: "principal"` and start through the deployment's principal-aware API with `external_user_id` in start params. `workspace` is valid for workspace-owned connections; `inherit` uses configured defaults. Control resolves once per instance and rejects conflicting explicit principal/workspace modes at principal resolution; dry-run reports `mcp_connection_resolution_conflict`. All uses of one instance should use a consistent resolution mode. Do not reuse one instance with conflicting principal/workspace modes.

Use `run.external_id` for the external principal key and `run.started_at` for run time. `run.run_id` is the execution id. Other recognized run fields include `external_user_id`, `principal_id`, `user_id`, `trigger_type`, and `segment_id`, but population depends on the start path. A fake principal/connection in trigger data does not resolve OAuth.

`connection` has `connection_id`, optional provider/account metadata, `by_instance`, and `primary_instance_id`. Root identity mirrors the resolved instance named by `primary_instance_id` (sorted-first), not the current step. With two or more MCP instances use `connection.by_instance["actual-instance-uuid"].connection_id` in Lua or `{{connection.by_instance.actual-instance-uuid.connection_id}}` in a template. Validate the entry exists before writing it as a foreign identity.

Bootstrap/event triggers must match actual provider/toolkit metadata. A Composio account event can identify its provider as `composio_account`; selecting "Any provider" may be too broad when more than one toolkit exists. Obtain the deployment's event/filter contract.

## Outputs and errors

Runtime exposes parsed MCP properties when available, otherwise structured content, otherwise raw output. Do not add a transport `structuredContent` layer unless it is present in the observed effective step result. For a documented Composio `{ successful, data, error }` envelope, fail on unsuccessful output before accessing data:

```lua
local function unwrap_composio(r)
  if type(r) ~= "table" then error("Missing Composio output") end
  if r.successful == false then error("Composio operation unsuccessful") end
  if r.error ~= nil and r.error ~= false and r.error ~= "" then
    error("Composio returned an error")
  end
  if r.successful ~= nil then
    if type(r.data) ~= "table" then error("Expected object Composio data") end
    return r.data
  end
  -- Use this branch only for a discovered normalized object-output tool.
  return r
end
local record = unwrap_composio(body.fetch)
if type(record.id) ~= "string" or record.id == "" then error("Missing resource id") end
return { resource_id = record.id }
```

Adapt object/scalar/array checks to the specific tool schema. Never unwrap an unsuccessful response into `{}` and treat it as an empty successful page. Field aliases must come from observed schemas, not a universal list of guesses. Treat resource ids as opaque; heuristic length/hex filters can silently discard legitimate data.

## Tenant Data contract

Obtain the **published** logical database schema, selected instance/database mapping, allowed operations, primary/unique keys, required columns, types, defaults, and table conflict/upsert policy. These are separate from connector OAuth. Scope shared customer data by the actual principal key; tenant authentication alone does not identify a customer row.

`tenantdata_insert.tool_args` contains `table`, `rows` (array of objects, maximum 100), and optional `on_duplicate`, `conflict_columns`, `upsert_columns`, `returning`. A whole-value template preserves a row object: `"rows": ["{{body.map_rows.record_row}}"]`. Read/update tool names and argument shapes must match the discovered tool schema.

Effective insert policy uses request overrides then table defaults. Conflict columns use a nonempty request override, then table default, then primary key; target must match the primary key or a single unique column. Upsert columns use a nonempty request override, then table default, then all non-conflict insert columns. An empty override does not disable a table's configured upsert list.

INSERT columns are the union of all row keys. Every effective upsert column must appear in that union and must not be a conflict column. In a mixed-key batch, a missing column value becomes SQL NULL for that row; it does not invoke a per-row default. For deterministic Lua upserts, supply every effective upsert key on every row with a type-valid non-nil value, or intentionally use a narrower nonempty override that matches the write's semantics.

Lua `nil` omits keys. It cannot emit an explicit SQL NULL key using a supported sentinel. This is a Lua limitation, not proof that all JSON/template/API callers cannot send null. For optional values prefer a deliberate omitted-column policy, narrower upsert override, or a separate update. Do not automatically overwrite unknown values with empty strings, false, zero, epoch time, or the run timestamp. Sentinels require a documented business convention.

| Logical type | SQL type | Authoring rule |
|---|---|---|
| string/email | text | Real text; empty only if semantically intended |
| short_text | varchar(255) | Respect length |
| integer/number | integer/numeric | Numeric values; no empty-string fallback |
| boolean | boolean | Native true/false; preserve false when selecting defaults |
| date/datetime | date/timestamptz | Valid date/ISO timestamp; never `datetime or ""` |
| uuid | uuid | Actual UUID; do not fabricate one for missing identity |
| json | jsonb | Preserve intended object/array shape; Lua empty table is object |

Audit fields should be explicit for MCP writes; do not assume REST auto-stamping applies. Lua filters with logical conjunction use `["and"]`, and writes should include the intended principal predicate. Multi-step writes are not one transaction: design idempotency and retry behavior before advancing state.
