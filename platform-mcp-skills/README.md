# AgentRuntime Platform MCP authoring skills

Plain Markdown skills that guide an AI agent through designing and building on AgentRuntime with the Platform MCP server: workspace data, workflows and apps.

| Skill | What it covers |
|-------|----------------|
| `agentruntime-platform-design` | Phased delivery: data, then workflows, then apps |
| `agentruntime-tenant-data-authoring` | Tenant Data schemas, migrations, history and upsert patterns |
| `agentruntime-workflow-authoring` | Workflow step JSON, validation and dry-run |
| `agentruntime-surfaces-authoring` | Surfaces v1 app, view and query JSON |

## Fetch (no auth)

Base URL: `https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/`

| Skill | SKILL.md |
|-------|----------|
| Platform design | [agentruntime-platform-design/SKILL.md](https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/agentruntime-platform-design/SKILL.md) |
| Tenant Data | [agentruntime-tenant-data-authoring/SKILL.md](https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/agentruntime-tenant-data-authoring/SKILL.md) |
| Workflows | [agentruntime-workflow-authoring/SKILL.md](https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/agentruntime-workflow-authoring/SKILL.md) |
| Surfaces | [agentruntime-surfaces-authoring/SKILL.md](https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/agentruntime-surfaces-authoring/SKILL.md) |

Index page: https://docs.agentruntime.io/api/platform-mcp-authoring-skills

The Platform MCP tool **`ar_get_started`** returns the same URLs.

## JSON schemas

Under `schemas/`. The Platform MCP tools `tenantdata_logical_schema` and `surfaces_schema_catalog` return them too.
