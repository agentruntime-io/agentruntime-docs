# Inputs an external agent needs

**Platform MCP discovery (2026-09):** `ar_get_started`, `catalog_connectors_browse`, `catalog_tools_search`, `catalog_tools_schema`, and `ar_builtin_tools_list` cover connector/tool/schema discovery for workflow authoring. See [platform-mcp-discovery.md](platform-mcp-discovery.md).

This checklist still applies for items not exposed as MCP tools (published Tenant Data logical schema, child workflow ids, deployment contract revision). Never invent an API path or resource id when no discovery capability is exposed.

| Required for | Obtain |
|---|---|
| Every workflow | Runtime/control/Studio versions or contract revision; supported step types; import/publish/start/validate API envelopes and authentication method |
| MCP | Tenant instance UUID, server/catalog id, tool input/output schemas, pagination/error envelope, credential ownership mode, rate/size limits |
| Embedded identity | Authorized external principal, resolved per-instance connection mapping or resolution capability, event provider/toolkit filters |
| Tenant Data | Instance-to-logical-database mapping, published schema/version, column constraints and conflict policy, permitted operations |
| Input | Workflow input schema, actual trigger payload with defaults materialized, per-step overrides if used |
| Child workflow | Published child id/version, accepted input keys, inherited identity behavior, `output_from` and result envelope, recursion/timeout limits |
| Run verification | Authorized test account/data, acceptable side effects, run status/events and error inspection capability |

A package for an agent without source access should include this skill folder, a deployment/version identifier, tool schemas with redacted representative outputs, database schema, binding manifest, and validation/start API examples. Supply secrets via the platform's authentication/connection mechanism, not workflow JSON or the bundle.

When data is missing, list the exact missing contract, keep placeholders visibly unbound, and finish graph work that does not depend on it. Report "authored, binding required" rather than "ready to run". If only dry-run access exists, report that connector and data behavior remain untested.
