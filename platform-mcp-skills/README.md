# Platform MCP authoring skills (public)

**Single source of truth** for external-agent skill markdown. Edit here only; do not maintain duplicate trees in the private platform-mcp repo or `.cursor/skills/`.

## Raw fetch (no auth)

Base: `https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/`

| Skill | SKILL.md |
|-------|----------|
| Platform design | [agentruntime-platform-design/SKILL.md](https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/agentruntime-platform-design/SKILL.md) |
| Tenant Data | [agentruntime-tenant-data-authoring/SKILL.md](https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/agentruntime-tenant-data-authoring/SKILL.md) |
| Workflows | [agentruntime-workflow-authoring/SKILL.md](https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/agentruntime-workflow-authoring/SKILL.md) |
| Surfaces | [agentruntime-surfaces-authoring/SKILL.md](https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/agentruntime-surfaces-authoring/SKILL.md) |

Human index: https://docs.agentruntime.io/api/platform-mcp-authoring-skills

Platform MCP **`ar_get_started`** returns the same raw URLs (`public_skill_urls.go` in agentruntime-platform-mcp).

## JSON schemas

Under `schemas/` — also returned by MCP `tenantdata_logical_schema` and `surfaces_schema_catalog`. When you change a schema file here, update matching **go:embed** copies in private `agentruntime-platform-mcp/tools/` before the next platform-mcp release.
