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
| [llm-step-results.md](references/llm-step-results.md) | `llm` step → `result.text`, JSON parse in Lua |
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
8. **Validate via server dry-run** (`workflows_validate` on Platform MCP) before running. JSON on disk is not saved anywhere: create or update the workflow to save it.

## Validation (server-first)

**Primary:** `POST /v1/workflows/{workflow_id}/validate` with inline `graph`, `params.trigger_payload`, and when using Tenant Data:

- `tenantdata_schema` — published logical schema, **or**
- Studio **Dry-run** — auto-fetches schema from bound Tenant Data MCP instances (read-only).

Optional request fields: `mapper_fixtures`, `mock_lua_scopes`, `mock_step_outputs`. See [dry-run.md](references/dry-run.md).

**Studio:** Import → map MCP + child workflows at Review import → Dry-run → Save → re-read graph. Saving updates Live, which runs at once; publish only when a caller must stay on a frozen copy.

## Live, published versions and what runs

Every workflow has one editable copy, **Live**, plus any number of frozen **published versions** (v1, v2, …).

| Do | How |
|----|-----|
| **Iterate on Live and run instantly** | `workflows_update` saves to Live. `workflow_run` / `workflows_execute_command` run Live by default, so the next run uses your edit. Nothing needs publishing to run, test, trigger, or be called as a child. |
| **Read `ran` to see what executed** | Run starts return `ran`: `{target: "live", revision}` or `{target: "published" \| "version", version}`. Quote it when reporting a run. |
| **Pass `base_revision` when editing** | Take `revision` from `workflows_get` (or your last `workflows_update`) and send it as `base_revision`. On `409 revision_conflict` nothing was saved: `workflows_get` again, re-apply your edit to the current graph, retry with the new revision. |
| **Publish plus pin when something must stay stable** | `workflows_publish_version` (optional `note`) freezes Live into the next version and returns `published_version`. Live stays editable. Publishing alone changes nothing that runs: callers opt in with `version: "published"` (newest published) or pin `version: "<n>"`. |
| **Pin other callers** | Inbound webhooks: `inbound_webhooks_create` / `inbound_webhooks_update` `version`. Child steps: `workflow` step `child_target` (`"published"` or a number; omit for Live; never author `child_revision`) — see [runtime-contract.md](references/runtime-contract.md#workflow-step-child-target). A published parent keeps its default children frozen at the revision they had when it was published. |
| **Look back** | `workflows_list_revisions` (Live history), `workflows_list_versions` (published versions only), `workflows_get` with `version: "<n>"` (read a frozen copy). |

`version: "published"` fails with `409 no_published_version` until the first publish; it never falls back to Live.

**No offline scripts in this bundle.** Use server dry-run / `workflows_validate` (see [validation.md](references/validation.md)).

## Reviews

Return an issue table: file, step id, exact path, rule, fix. Distinguish confirmed failures, conditional risks, and checks not run. Do not claim validate/run/publish without evidence (`checks_performed`; run id plus `ran`; `published_version` when you published).

## Import

Resolve `REPLACE_*` MCP instance placeholders and `REPLACE_CHILD_*` child `workflow_id` values at Studio **Review import**. Children need no publishing first: create or import them, then reference their ids; a parent running Live calls each child's Live. Workflow JSON is source of truth — no patch generators.
