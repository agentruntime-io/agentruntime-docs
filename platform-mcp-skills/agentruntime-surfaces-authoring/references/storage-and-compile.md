# Surfaces storage and compile

See monorepo **`docs/surfaces/SURFACES_STORAGE.md`** (canonical).

## Persisted artifacts

| JSON file | Storage |
|-----------|---------|
| Thin `app.json` | `surface_apps.manifest` |
| `*-view.json` | `surface_app_views.document` |
| `*.query.json` | `surface_app_queries.document` |
| `marketplace-package.json` | `app_packages.manifest` |

Workflow graphs stay in **workflow_definitions** — Surfaces only holds **UUID refs** and bindings.

## Runtime data flow

| Binding `source` | Backing store |
|------------------|---------------|
| `workflow.node.output` | agentruntime run/step state |
| `database.query` | tenant-data-service SELECT |
| `connection.read` | BFF → MCP read tools |
| `tenant.config` | tenant settings |

## Compile / import pending

App Studio may report **`import_pending`** until workflow UUIDs in the view doc resolve to published workflows. Agents should:

1. Publish workflow (Console or `workflows_publish_version`).
2. Replace placeholder refs with real UUIDs (`replaceWorkflowRefs` pattern in webapp).
3. Re-save view document via **`surfaces_apps_put_view`**.
4. Read **`bind_compile`** on **`surfaces_apps_get_view`** — do not skip step 4.

**Saved views ≠ working apps.** See [app-studio-wiring.md](app-studio-wiring.md) for attached workflows, query data sources, run/approve actions, and **`surfaces_render`** verification.

## Files vs Tenant Data in UI

Use **Work files** for large uploads; Surfaces can link file pickers through workflow inputs. Structured extraction output belongs in **Tenant Data** (tenant-data skill).
