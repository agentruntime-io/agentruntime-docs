# Bundled examples

`scope-smoke.json` is a connector-free graph demonstrating two separate top-level composite loops, raw inner Lua paths, table-return while evaluation, and an explicitly wired downstream read. Expected totals: two for_each items; while ends after two iterations with `condition_met`.

Workflow JSON is the source of truth. Keep scripts inline in that JSON; do not depend on an integration-specific patch generator. Combine the scope pattern with the deployment's actual MCP schemas and Tenant Data contract. For composed workflows, import/publish child workflows first and map parent `REPLACE_CHILD_*` references at Studio Review import.
