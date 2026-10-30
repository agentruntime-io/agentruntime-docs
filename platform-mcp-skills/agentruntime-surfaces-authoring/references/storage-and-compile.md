# Surfaces: what is saved, and compile

## What each artifact is

| JSON | Saved by | Notes |
|------|----------|-------|
| Thin app manifest (`app.json`) | `surfaces_apps_create` / `surfaces_apps_update` | Metadata + `navigation` only |
| View document (`*-view.json`) | `surfaces_apps_put_view` | One per `view_id`; the view body is the source of truth |
| Query document (`*.query.json`) | `surfaces_apps_put_query` | Read-only parameterised SQL for `database.query` |
| Marketplace package | `surfaces_apps_publish_package` | Catalog copy of the app |

Workflow graphs are not part of an app. A view only holds **workflow UUID refs** and bindings to them.

## Where data comes from at runtime

| Binding `source` | Backing data |
|------------------|--------------|
| `workflow.node.output` / `workflow.node.state` / `workflow.run` | The workflow run's step results and state |
| `database.query` | Read-only `SELECT` against the workspace's Tenant Data databases |
| `connection.read` | A read tool on a connected service |
| `tenant.config` | Workspace settings |

## Compile / import pending

Every `surfaces_apps_put_view` checks the view's workflow refs against the workflow's Live copy (no publishing needed) and stores the result as `bind_compile` on the view. `import_pending` means a `workflow_ref` is not yet a UUID of an existing workflow. To resolve it:

1. Create or update the workflow (`workflows_create` or `workflows_update`).
2. Replace placeholder refs in the view document with the real workflow UUIDs.
3. Re-save the view with **`surfaces_apps_put_view`**.
4. Read **`bind_compile`** from **`surfaces_apps_get_view`**. Do not skip this step.

A view with broken bindings still saves; its actions are blocked at render/action time until the bindings compile.

**Saved views ≠ working apps.** See [app-studio-wiring.md](app-studio-wiring.md) for attached workflows, query data sources, run/approve actions, and **`surfaces_render`** verification.

## Files vs Tenant Data in the UI

Use **Work files** for large uploads; Surfaces can link file pickers through workflow inputs. Structured extraction output belongs in **Tenant Data** (tenant-data skill).
