# Runtime contract

This is a portable authoring profile of the reviewed Go runtime. The importer accepts other forms, including canvas JSON, but external agents should produce `steps[]`. The bundled schema is a conservative profile, not a generated specification of all runtime fields. JSON Schema validation and static lint are different checks.

## Graph and scope

Every step needs an id and type; give it a descriptive name. Top-level ids are unique. Inner ids are unique per parent and must not collide with top-level ids; different parents may reuse an inner id. Runtime accepts simple identifiers such as `fetch_page`. Studio converts non-UUID top-level ids to UUIDs and rewrites recognized template/Lua references; dynamically constructed references are not a safe remapping contract. For Studio-bound artifacts, stable UUIDs plus Lua `steps["uuid"]` avoid that migration. Verify stored ids after import rather than assuming slugs survive.

Top-level `depends_on` gives execution order, not just visual edges. Every top-level template reference must name a transitive ancestor. A nonempty `input_allowlist` of step ids is an additional restriction, never an alternative dependency. Lua reads are not inferred reliably by dry-run: wire them yourself.

Inner steps may be `mcp_call`, `lua_script`, `llm_call`, or `agent_call`. No nested composites, `no_auto_start`, or routing out of a body. Put upstream top-level dependencies on the parent; use inner ids for inner ordering. Runtime preserves array order when no inner dependencies exist and otherwise performs a stable topological sort. Prefer explicit inner chains.

| Value | Template | Lua |
|---|---|---|
| Trigger input | `{{input.topic}}` | `input.topic` |
| Completed top-level output | `{{steps.fetch.result.data}}` | `steps.fetch.result.data` |
| Resolved top-level input | `{{steps.fetch.input.field}}` | `steps.fetch.input.field` |
| Current inner output | `{{body.fetch.data}}` | `body.fetch.data` |
| Current for_each item | `{{item.id}}` / `{{foreach.item.id}}` | `item.id` / `foreach.item.id` |
| for_each index, zero-based | `{{index}}` | `index` |
| Canonical Studio item alias | `{{steps.each.item.id}}` inside `each` | Prefer `item.id` |
| while iteration, zero-based | `{{while.iteration}}` | `loop.iteration` |
| Previous while body output | `{{while.prev.fetch.data}}` | `loop.prev.fetch.data` |
| Current while body output, including eval | `{{body.score.pass}}` | `body.score.pass` |

Template compatibility supports an optional `.result` segment on `body`/`while.prev`; Lua does not add that wrapper. If a tool actually has a field called `result`, access according to its documented output schema, not a guessed wrapper. Templates can ambiguously traverse a real `result` field, so prefer direct body paths for ordinary fields.

Guard missing previous data on iteration zero:

```lua
local prev = (loop.prev and loop.prev.capture) or {}
return { page_token = prev.next_page_token or "" }
```

## Lua

Return a keyed table; top-level sequence tables serialize as maps of numeric-string keys. Nested nonempty sequences serialize as arrays. An empty Lua `{}` serializes as `{}`, not `[]`. Assigning `nil` removes a key. There is no supported Lua null sentinel or distinct empty-array constructor in this snapshot.

Use native booleans/numbers and reserved-key syntax `["and"] = {...}`. Do not put `{{...}}` in Lua source: scripts execute directly, without template substitution. `input`, `steps`, `run`, and `connection` are injected; `item`, `foreach`, `body`, `loop` are meaningful only in their scopes. Safe basic/string/table/math functions exist; no OS, network, module loading, or filesystem access. Script limit 64 KiB in UTF-8 bytes; result limit 1 MiB. Default timeout 5 seconds, maximum 30 seconds even if a larger `timeout_s` is authored.

## for_each

Fields: `for_each_items` string, nonempty `for_each_body_steps`, optional `for_each_max_parallel` (default 10, maximum 100), `for_each_error_policy` (`fail_fast`, `continue_on_error`, `pause`, `human_review`). Maximum 500 items per loop invocation.

Produce a present list field, including the empty case: `return { items = items }`. Empty Lua `{}` is accepted as an empty iteration list. **Do not omit the field by returning nil:** unresolved mustache strings now fail instead of becoming a bogus singleton item. A resolved null or empty array is empty, but a missing path is not resolved null. Runtime also coerces scalars/non-array maps to singleton lists; that is compatibility behavior, not a recommended authoring pattern.

Completed parent result contains `count`, `max_parallel`, `succeeded_count`, `failed_count`, and `results`. Each successful entry contains `index`, raw `body` outputs, and `result` from the selected last body step. Prefer named `results[i].body.map_rows` when aggregating. Failed entries contain `index` and `error`. `continue_on_error` can complete with failures and `partial_success`; check the counts before checkpointing. `fail_fast` makes the wrapper fail but is not transactional and does not guarantee already scheduled side effects stop.

## while

Fields: `while_mode` (`until` or `while`), positive `while_max_iterations`, nonempty `while_body_steps`, and `while_condition` or `while_eval_script`. The body executes before evaluation, including the first iteration. If both evaluation fields exist, Lua wins.

```lua
return { pass = body.capture.no_more_pages == true }
```

Runtime reads `pass` (or `result` if `pass` is absent). Use an actual boolean. `until` stops on true; `while` stops on false. Scalar `return true` compiles but fails execution because all workflow Lua must return a table.

Result fields: `iteration_count`, `max_iterations`, `stopped_reason`, `last`, `iterations`. `last.capture` is raw, so downstream Lua is `steps.paginate.result.last.capture.next_page_token`. A cap returns `stopped_reason = "max_iterations"` without necessarily failing the step. Inspect it before treating pagination as complete. Large accumulated pages and retained iterations can exceed Lua/result or downstream list limits: choose bounded batches and continuation, not unbounded accumulation.

## Other step types

| Type | Runtime configuration minimum |
|---|---|
| `mcp_call` | `tool_name`; resolvable `instance_id` or `server_url`; schema-correct `tool_args` |
| `lua_script` | Nonempty `script` |
| `llm_call` | `model` or `model_ref`, nonempty `prompt`; obtain provider/credential contract — see [llm-step-results.md](references/llm-step-results.md) |
| `agent_call` | `agent_id`, nonempty `prompt` or `input`; obtain agent schema |
| `human_task` | `task_type`; obtain payload/response contract for that task type |
| `workflow_call` | Child `workflow_id`, not parent id; obtain child published version/input/output contract |
| `external_agent` | Deployment-specific provider/configuration contract required |

Advanced routing and child-run semantics require deployment documentation; the bundled schema does not certify these complete configurations.

## Template pipe modifiers (optional)

Inside one `{{ ... }}` expression, `|` means else-try left to right. Trailing `omit` drops the key from resolved step input when unset; trailing JSON literal supplies a default; path segments coalesce (`{{input.a | steps.read.result.row.a | omit}}`). Applies to `workflow_call` child input and other template-expanded fields. See [template-modifiers.md](template-modifiers.md).

## Connection context and diagnostics

`connection.primary_instance_id` identifies the sorted-first resolved instance whose identity is mirrored by root `connection.connection_id`, `provider`, `provider_account_key`, and `metadata`. It is not the current step's instance. With two or more MCP instances use explicit `connection.by_instance.<uuid>.*` templates or Lua `connection.by_instance["uuid"]` for identity. See [connectors-and-data.md](connectors-and-data.md).

| Dry-run code | Meaning / response |
|---|---|
| `connection_root_multi_instance` | Warning: root identity used with 2+ MCP instances. Select the intended `by_instance` entry. |
| `for_each_items_unresolved` | Error: unresolved mustache survived list resolution. Supply the missing input/path; never treat it as an item. Current source may instead report `for_each_items_invalid` from the coercion guard. |
| `mcp_connection_resolution_conflict` | Error: one instance requests conflicting explicit principal/workspace modes. Use consistent binding modes. |
| `mcp_connection_resolution_principal` | Warning for a Composio principal-bound step: principal context is required at run time; this is not a request to change it to workspace. |

The current compile-only whitelist does not yet include `connection.primary_instance_id`, although Control exposes it at run start. Dry-run may falsely flag its template without live context; do not invent a primary id to silence that diagnostic. See [validation.md](validation.md) for remaining check-registration and loop-validation gaps.
