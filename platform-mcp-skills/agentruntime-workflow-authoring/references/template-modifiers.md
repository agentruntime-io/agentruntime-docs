# Template pipe modifiers

Optional syntax inside a single `{{ ... }}` expression. Read `|` left-to-right as **else try**. The last segment sets policy when nothing before it resolved.

## Syntax

| Pattern | Behavior |
|---|---|
| `{{input.field}}` | Required — fail if missing or unresolved |
| `{{input.field \| "2026/01/01"}}` | Default literal when field is missing or empty string |
| `{{input.field \| 120}}` | Default number (JSON literal) |
| `{{input.field \| omit}}` | Drop the key from the resolved map (child never sees it) |
| `{{input.a \| steps.read.result.row.a \| omit}}` | Coalesce: try `input.a`, else step output, else omit key |

**Unset** for coalesce/default: missing path, `nil`, or `""` (empty string). A bare required template still passes through empty string unchanged.

Quotes inside literals use JSON rules: `{{input.q | "(in:inbox OR in:sent) after:2026/01/01"}}`.

## Where it applies

Any step field resolved through template expansion: `workflow_call` child input, `mcp_call` `tool_args`, `agent_call` input, etc. Omitted keys are not sent to the child/tool payload.

## Authoring examples (workflow_call)

```json
"input": {
  "backfill_after": "{{input.backfill_after | \"2026/01/01\"}}",
  "backfill_query": "{{input.backfill_query | omit}}",
  "poll_interval_seconds": "{{input.poll_interval_seconds | 120}}",
  "backfill_page_token": "{{input.backfill_page_token | steps.read_sync_state.result.row.backfill_page_token | omit}}"
}
```

Studio **Child inputs** stays a plain template field per schema property — type modifiers directly or import JSON. No separate Required/Default/Omit UI.

## Dry-run

Dry-run treats `| omit` and trailing literal defaults as satisfiable without live values. Required input-only chains (no trailing policy) still fail when no segment resolves. `workflow_call` child `input` templates are included in the server `templates` check. Dependency reachability for `steps.*` segments in coalesce chains is validated by server `dependencies` / `templates` checks.
