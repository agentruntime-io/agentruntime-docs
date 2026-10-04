# Surfaces v1 — machine schema catalog

**First:** call Platform MCP **`surfaces_schema_catalog`** — primary URLs point at **`agentruntime-platform-mcp/tools/surfaces-schemas/`** (no monorepo access required).

Monorepo mirror (optional, may 404 without repo access):

| Artifact | Schema entrypoint | Example |
|----------|-------------------|---------|
| **App** | `app.schema.json` → `surfaces-v1#/$defs/AppDocument` | `examples/tiktok/app.json` |
| **View** | `view.schema.json` | `examples/tiktok/publish-view.json` |
| **Query** | `query-definition.schema.json` | `examples/tiktok/pending-posts.query.json` |
| **Marketplace package** | `marketplace-package.schema.json` | `examples/tiktok/marketplace-package.json` |
| **Render request/response** | `render-request.schema.json`, `render-response.schema.json` | API-only |

Base URL:

`https://raw.githubusercontent.com/agentruntime-io/agentruntime/develop/docs/surfaces/schemas/`

Files:

- `surfaces-v1.schema.json` — shared `$defs`
- `app.schema.json`, `view.schema.json`, `query-definition.schema.json`
- `marketplace-package.schema.json`

Human docs: `docs/surfaces/SURFACES_FIELD_REFERENCE.md`, `SURFACES_RUNTIME_CONTRACTS.md`.
