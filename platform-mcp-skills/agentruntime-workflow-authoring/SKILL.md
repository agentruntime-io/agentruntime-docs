---
name: agentruntime-workflow-authoring
description: Author and review AgentRuntime steps[] workflow JSON, including Lua, composite loops, MCP bindings, and Tenant Data writes, using a portable runtime contract. Use for import, dry-run, and execution problems; obtain tenant tool schemas and bindings rather than inventing them.
---

Produce a workflow plus its binding/input requirements and validation evidence. This bundle works without repository access. It describes the source snapshot in [provenance.json](references/provenance.json), not every deployed release. If deployment capabilities differ, use the deployment's contract and report the difference.

## Hosted copy (external agents)

**Browse:** https://github.com/agentruntime-io/agentruntime-docs/tree/main/platform-mcp-skills/agentruntime-workflow-authoring

**Fetch SKILL.md:** https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/agentruntime-workflow-authoring/SKILL.md

Platform MCP clients: call **`ar_get_started` first** — it returns skill URLs in `documentation`, **`recommended_next_actions`** playbooks, `documentation.embedded_schema_tools`, and `platform_mcp.publishable_tool_count`. Connector/tool discovery: [platform-mcp-discovery.md](references/platform-mcp-discovery.md). Validate before run: [dry-run.md](references/dry-run.md).

## Read first

| File | When |
|------|------|
| [platform-mcp-discovery.md](references/platform-mcp-discovery.md) | **Platform MCP** — `ar_get_started`, catalog browse/search/schema, builtins |
| [runtime-contract.md](references/runtime-contract.md) | Always — graph, Lua, loops, templates |
| [dry-run.md](references/dry-run.md) | Always — server validate API and checks |
| [validation.md](references/validation.md) | Handoff, Studio flow, request bodies |
| [template-modifiers.md](references/template-modifiers.md) | Optional `{{input \| …}}` pipe syntax |
| [connectors-and-data.md](references/connectors-and-data.md) | MCP + Tenant Data writes |
| [llm-step-results.md](references/llm-step-results.md) | `llm_call` → `result.text`, JSON parse in Lua |
| [deployment-inputs.md](references/deployment-inputs.md) | Bindings, tenant export, run setup |
| **GET `/api/v1/workflows/schema`** (agentruntime) — canonical step JSON Schema (`workflow/schema.go`) | Always before authoring step bodies |
| **`workflows_graph_schema`** (Platform MCP) — same schema + `schema_hash` via BFF | Prefer on Platform MCP instead of raw agentruntime URL |
| **`workflows_authoring_contract`** (Platform MCP) | Call when tempted to use nodes/edges JSON — runtime uses `graph.steps[]` only |
| [schemas/studio-workflow-import.schema.json](schemas/studio-workflow-import.schema.json) | Import envelope + curated subset |
| [examples/patterns.md](examples/patterns.md) | Portable patterns |

## Authoring rules (summary)

1. **Platform MCP:** `ar_get_started` → `catalog_connectors_browse` → `catalog_tools_search` → `catalog_tools_schema` → `ar_builtin_tools_list`. Do not invent tool names or `tool_args` keys.
2. Obtain remaining deployment inputs (published Tenant Data schema, child workflow ids). Mark unresolved `REPLACE_*` bindings explicitly.
3. Author `steps[]` with unique top-level ids and acyclic `depends_on`. Inner ids are scoped per composite body.
4. Inner Lua: `body.inner_id` (raw). While previous iteration: `loop.prev.inner_id`. Top-level Lua: `steps.step_id.result`. Do not add `.result` inside composite Lua paths.
5. No nested `for_each` / `while` in composite bodies.
6. All Lua returns a table; `while_eval_script` returns `{ pass = boolean }`.
7. Do not advance cursors after partial failure or pagination cap without durable recovery.
8. **Validate via server dry-run** (`workflows_validate` on Platform MCP) before claiming runnable. JSON on disk does not publish.

## Validation (server-first)

**Primary:** `POST /v1/workflows/{workflow_id}/validate` with inline `graph`, `params.trigger_payload`, and when using Tenant Data:

- `tenantdata_schema` — published logical schema, **or**
- Studio **Dry-run** — auto-fetches schema from bound Tenant Data MCP instances (read-only).

Optional request fields: `mapper_fixtures`, `mock_lua_scopes`, `mock_step_outputs`. See [dry-run.md](references/dry-run.md).

**Studio:** Import → map MCP + child workflows at Review import → Dry-run → Save → Publish → re-read graph.

**No offline scripts in this bundle.** Use server dry-run / `workflows_validate` (see [validation.md](references/validation.md)).

## Reviews

Return an issue table: file, step id, exact path, rule, fix. Distinguish confirmed failures, conditional risks, and checks not run. Do not claim validate/publish/run without evidence (`checks_performed`, published version id, run id).

## Import

Resolve `REPLACE_*` MCP instance placeholders and `REPLACE_CHILD_*` child `workflow_id` values at Studio **Review import**. Publish children before orchestrators. Workflow JSON is source of truth — no patch generators.
