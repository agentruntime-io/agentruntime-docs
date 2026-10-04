# Platform MCP — discovery and authoring bootstrap

External agents connected to **AgentRuntime Platform MCP** (`mcp.agentruntime.io` or local `8340/mcp`) should use typed catalog tools — not raw `catalog_tools_list` dumps or `catalog_bff_request` unless debugging.

**Hosted skill (read in browser):**  
https://github.com/agentruntime-io/agentruntime-docs/tree/main/platform-mcp-skills/agentruntime-workflow-authoring

**Skill entry (fetch as markdown):**  
https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/agentruntime-workflow-authoring/SKILL.md

`ar_get_started` returns skill URLs in `documentation`, **`recommended_next_actions`** playbooks, `documentation.embedded_schema_tools`, and `platform_mcp.publishable_tool_count`.

---

## Call order

```
ar_get_started
  → tenantdata_logical_schema       (when designing tables)
  → tenantdata_databases_create | tenantdata_schema_migrate*
  → tenantdata_mcp_provision
  → workflows_authoring_contract
  → workflows_graph_schema
  → workflows_llm_step_contract    (when graph uses llm_call)
  → catalog_connectors_browse     (which connectors exist; ready vs needs wire vs catalog)
  → catalog_tools_search          (tool names + instance_id per connector)
  → catalog_tools_schema          (input_schema / output_schema for tool_args)
  → ar_builtin_tools_list         (lua_script, tenantdata_*, etc.)
  → workflows_graph_schema
  → workflows_create / workflows_update
  → workflows_validate
  → workflow_run (+ workflow_await / workflow_inspect as needed)
  → surfaces_schema_catalog       (when authoring App Studio JSON)
```

| Step | Tool | You get |
|------|------|---------|
| 0 | `ar_get_started` | Tenant, project, capabilities, playbooks, documentation URLs |
| 0b | `tenantdata_*` | Logical schema meta, apply schema, api-contract, MCP instance |
| 1 | `catalog_connectors_browse` | `groups.ready` / `needs_connection` / `catalog` — slug → tool count |
| 2 | `catalog_tools_search` | `connectors[]` with `tools[]` (name, description) and `instances[]` (instance_id, has_connection) |
| 3 | `catalog_tools_schema` | `input_schema`, `output_schema` per tool name (batch, max 20; no instance_id — schemas are per server) |
| 4 | `ar_builtin_tools_list` | Builtin step types and field lists (not connector MCP tools) |
| 5 | `workflows_*` | Create/update graph, validate, run |

Follow `recommended_next_actions` from `ar_get_started` when unsure which playbook applies.

---

## `catalog_connectors_browse`

```json
{ "page": 1, "page_size": 100 }
```

- Paginates **connector slugs** (not tools).
- `groups.ready.connectors` — installed + connected; safe to plan `mcp_call` steps.
- `groups.needs_connection` — instance exists but not wired.
- `groups.catalog` — not installed.

---

## `catalog_tools_search`

At least one of `tools` or `query` is required.

```json
{
  "tools": ["gmail_send_message"],
  "query": "send email",
  "page": 1,
  "page_size": 100
}
```

- **Instances are per connector**, not per tool — all tools on Gmail share the same `instances[]`.
- Pick `instance_id` from `instances[]` where `has_connection: true` when possible.
- Paginates over **tools**; results grouped by `connector` on each page.

---

## `catalog_tools_schema`

```json
{ "tools": ["gmail_send_message", "slack_post_message"] }
```

- 1–20 exact tool names from search.
- **No `instance_id`** — schemas do not vary by instance.
- Map `name` → `tool_name` on `mcp_call` steps; build `tool_args` from `input_schema.properties`.
- Use `output_schema` for `{{steps.<id>.result.*}}` wiring.
- Partial success: `tools[]` + `errors[]` (`not_found`, `schema_unavailable`, `server_detail_failed`).

Example `mcp_call` step:

```json
{
  "id": "send_email",
  "type": "mcp_call",
  "tool_name": "gmail_send_message",
  "instance_id": "<from catalog_tools_search instances[]>",
  "tool_args": {
    "to": "{{input.recipient}}",
    "subject": "{{input.subject}}"
  },
  "depends_on": []
}
```

---

## Validation and run

**Rule:** never call `workflow_run` on a new or edited graph until `workflows_validate` returns `valid: true`. Dry-run checks structure, templates, Lua, and MCP bindings — it does **not** call live connectors or SQL.

| Goal | Tool |
|------|------|
| Dry-run compile check | `workflows_validate` |
| Start + wait | `workflow_run({ workflow_id, wait: true })` |
| Start background | `workflow_run({ workflow_id, wait: false })` → `workflow_await` |
| Approval gate | `workflow_inspect` → user confirms → `workflow_approve` |

### `workflows_validate` example

Use the draft `workflow_id` from `workflows_create` / `workflows_get`. Inline `graph` is required.

**Step fields are flat on each step** — do not nest `model` / `prompt` under `config` (that shape is only for `external_agent`).

```json
{
  "workflow_id": "<uuid>",
  "graph": {
    "input_schema": [
      { "name": "message_text", "type": "string", "optional": false }
    ],
    "steps": [
      {
        "id": "generate_reply",
        "type": "llm_call",
        "model": "gpt-4o-mini",
        "model_ref": "direct.openai.gpt-4o-mini",
        "prompt": "Reply to: {{input.message_text}}",
        "depends_on": []
      }
    ]
  },
  "params": {
    "trigger_payload": {
      "message_text": "Hi, can we meet tomorrow?"
    }
  }
}
```

Put start inputs under **`params.trigger_payload`** (not a bare top-level `payload` object) so `{{input.*}}` templates resolve.

| Response field | Meaning |
|----------------|---------|
| `valid: true` | Safe to attempt `workflow_run` (still not proof of live MCP/SQL success) |
| `valid: false` | Fix `issues[]` before run — never treat empty `{}` as success |
| `issues[]` | Each: `severity`, `code`, `step_id`, `message` |
| `checks_performed[]` | Server checks that ran |
| `topological_order[]` | Step execution order used for dry-run simulation |

Full check list, issue codes, and limits: [dry-run.md](dry-run.md). Graph rules: [SKILL.md](../SKILL.md), [runtime-contract.md](runtime-contract.md).

---

## Avoid for agent authoring

| Tool | Why |
|------|-----|
| `catalog_tools_list` | Full catalog dump (~1500 rows), no schemas |
| `catalog_bff_request` | Held / debug multiplexer |
| `catalog_servers_get` | Whole server payload when you only need a few tool schemas |

---

## After discovery

Continue with [connectors-and-data.md](connectors-and-data.md) (identity, Composio unwrap, Tenant Data) and [runtime-contract.md](runtime-contract.md) (graph shape, Lua, loops).
