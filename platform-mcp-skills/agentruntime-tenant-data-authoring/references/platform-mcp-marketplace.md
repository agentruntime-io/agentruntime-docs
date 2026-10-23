# Platform MCP — marketplace packages

| Tool | Purpose |
|------|---------|
| `marketplace_install_plan` | Preflight install (issues, binding keys) |
| `marketplace_install_apply` | Install with `resolutions` |
| `marketplace_database_packages_publish` | Publish schema recipe from manifest |
| `marketplace_workflow_packages_publish` | Snapshot the workflow's newest published version (publishing Live first if it changed) to workflow_packages |
| `marketplace_database_packages_list` | Browse database packages (open catalogs + your own org listings) |
| `marketplace_database_packages_lookup` | Fetch one package + manifest by exact id |

Install `entry_kind`: `agent`, `workflow`, `database`, `bundle`, `app`.

**Publishing a workflow package:**
- `package_id` must start with your workspace prefix: the first 8 characters of your tenant id plus `/` (e.g. `1a2b3c4d/order-intake`). Otherwise `400 package_namespace`; `403 package_owned_by_another_tenant` when another workspace already owns that id and version.
- The package `version` is a package label (default `1`), separate from workflow version numbers.
- The source workflow stays a normal, visible workflow.
- Visibility: `private` (default; never listed to others, readable by exact id), `org` (listed to your workspace only), `public`.

**Installed copies:** `workflows_get` on an installed workflow returns `source_package_id`, `source_package_version`, `latest_package_version` and `update_available`.
