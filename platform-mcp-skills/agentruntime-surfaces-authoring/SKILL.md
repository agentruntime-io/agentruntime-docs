---
name: agentruntime-surfaces-authoring
description: Author Surfaces v1 app/view/query JSON for Space/Console apps, marketplace packages, and database.query bindings. Use after workflows and Tenant Data exist.
---

Design **UI shells** stored in control-service (`surface_apps`, `surface_app_views`, `surface_app_queries`) — not workflow graphs.

## Hosted copy (external agents)

**Browse:** https://github.com/agentruntime-io/agentruntime-docs/tree/main/platform-mcp-skills/agentruntime-surfaces-authoring

**Fetch SKILL.md:** https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/agentruntime-surfaces-authoring/SKILL.md

**Schema catalog:** Platform MCP **`surfaces_schema_catalog`** or monorepo `docs/surfaces/schemas/`.

Platform MCP: **`ar_get_started`** → `documentation.surfaces_authoring_skill_*`.

## Read first

| Resource | Purpose |
|----------|---------|
| [references/app-studio-wiring.md](references/app-studio-wiring.md) | **Required for MCP agents** — attached workflows, query data sources, button actions, `bind_compile` verification (no separate “attach” tool) |
| [references/schema-catalog.md](references/schema-catalog.md) | JSON Schema URLs + artifact types |
| [references/storage-and-compile.md](references/storage-and-compile.md) | What persists where; workflow UUID refs |
| Monorepo `docs/surfaces/SURFACES_SCHEMA.md` | Human spec + validate script |
| Monorepo `docs/surfaces/examples/tiktok/` | End-to-end sample |
| [agentruntime-workflow-authoring](../agentruntime-workflow-authoring/SKILL.md) | Workflow refs must be **published workflow UUIDs** |
| [agentruntime-tenant-data-authoring](../agentruntime-tenant-data-authoring/SKILL.md) | `database.query` targets Tenant Data |

## Authoring rules (summary)

1. **`workflow_ref`** fields are **UUID only** (published `workflow_definitions.id`) — no slugs in saved artifacts.
2. App manifest (`app.json`) is **thin**: metadata + `navigation` — views live in separate `*-view.json` files.
3. Component props: literals inline; dynamic values use `{ "bind": { "source": … } }` (see SURFACES_SCHEMA.md).
4. **`database.query`** binds read-only SELECT to Tenant Data queries defined in `*.query.json` — **`surfaces_apps_put_query` before** the view references `query_id`.
5. **Runnable apps** need **`workflow.run` / `workflow.operation` actions**, not only display bindings — see [app-studio-wiring.md](references/app-studio-wiring.md).
6. Publish: **`workflows_publish_version`** then optional **`marketplace_workflow_packages_publish`**. App: **`surfaces_apps_*`** + **`surfaces_apps_publish_package`**. HITL: **`surfaces_apps_put_gate_bindings`**.
7. After every view save, confirm **`bind_compile.status`** via **`surfaces_apps_get_view`** (and app-level status via **`surfaces_apps_get`**).

Validate JSON locally: `cd docs/surfaces/schemas && node validate-examples.mjs` (monorepo).
