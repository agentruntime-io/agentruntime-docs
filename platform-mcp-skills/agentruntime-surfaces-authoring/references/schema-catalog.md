# Surfaces v1 — schema catalog and document shapes

**First:** call Platform MCP **`surfaces_schema_catalog`** — it returns the schema URLs below.

Base URL (public, no auth):

`https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/schemas/surfaces/`

| Artifact | Schema file | Saved with |
|----------|-------------|------------|
| **App** (thin manifest) | `app.schema.json` → `surfaces-v1.schema.json#/$defs/AppDocument` | `surfaces_apps_create` / `surfaces_apps_update` (`manifest`) |
| **View** | `view.schema.json` | `surfaces_apps_put_view` (`document`) |
| **Query** | `query-definition.schema.json` | `surfaces_apps_put_query` (`document`) |
| **Marketplace package** | `marketplace-package.schema.json` | `surfaces_apps_publish_package` |

`surfaces-v1.schema.json` holds the shared `$defs` the others reference. Render request/response and action request are API payloads (see [app-studio-wiring.md](app-studio-wiring.md)); there is no schema file for them here.

## App document (thin)

Metadata and navigation only — view bodies are saved separately.

```json
{
  "schema_version": "1",
  "id": "tiktok",
  "name": "TikTok",
  "description": "Review and publish videos to TikTok.",
  "icon": "tiktok",
  "default_view_id": "publish",
  "navigation": [
    { "label": "Publish", "view_id": "publish" },
    { "label": "Queue", "view_id": "queue" }
  ]
}
```

## View document

Required: `schema_version`, `id`, `root`. Optional: `name`, `workflows[]` (`{ "workflow_ref": "<uuid>", "role": "primary" | "data" | "action" }`; `role` is a Studio hint only), `root` = component tree (`id`, `kind`, `props`, `children`, `events`).

## Prop values: literals vs bindings

Static values are literals. Dynamic values use `{ "bind": … }`:

```json
"props": {
  "label": "Caption",
  "rows": 4,
  "value": {
    "bind": {
      "source": "workflow.node.output",
      "workflow_ref": "a1b2c3d4-e5f6-4789-a012-3456789abcde",
      "node_id": "generate_caption",
      "path": "result.text"
    }
  }
}
```

Do not wrap static values in `{ "bind": { "source": "static", … } }`.

### Binding sources (read-only)

| `source` | Required fields |
|----------|-----------------|
| `static` | `value` |
| `workflow.node.output` | `workflow_ref` (UUID), `node_id`, `path` |
| `workflow.node.state` | `workflow_ref` (UUID), `node_id`, `path` |
| `workflow.run` | `workflow_ref` (UUID), `path` |
| `database.query` | `query_id`, optional `params` |
| `connection.read` | `connection_id`, `tool`, optional `args` |
| `tenant.config` | `key` |

Optional `transform`: a string such as `"eq:completed"` / `"neq:pending"`, or `{ "op": "eq", "value": "completed" }`.

## Editable fields (human review)

On input components (`TextField`, `TextArea`, `Select` use `value`; `Checkbox`, `Switch` use `checked`), set in `props`:

| Field | Purpose |
|-------|---------|
| `editable` | `true` → local draft until an action fires |
| `context_edit` | `{ "step_id": "…", "path": "…", "format": "text" }` (`format` optional) → submitted as a context edit when the review step is approved |

## Actions (writes)

Writes happen only through actions, wired under `events.onClick`, `onSubmit`, `onChange` or `onSelectionAction` as `{ "action": { … } }`.

| `type` | Fields |
|--------|--------|
| `workflow.operation` | `workflow_ref`, `node_id`, `operation` (e.g. `approve`, `reject`), optional `collect_context_edits` |
| `workflow.run` | `workflow_ref`, optional `params` (supports `{ "from": "selection", "path": "id" }`), optional `collect_workflow_inputs` |
| `workflow.cancel` | `workflow_ref`, `run_id` |

Approve button at a review step:

```json
"events": {
  "onClick": {
    "action": {
      "type": "workflow.operation",
      "workflow_ref": "a1b2c3d4-e5f6-4789-a012-3456789abcde",
      "node_id": "review_gate",
      "operation": "approve",
      "collect_context_edits": true
    }
  }
}
```

## Query document

Required: `schema_version`, `id`, `sql`. Optional: `name`, `params` (map of name → `{ "type": …, "default": … }`). SQL is read-only `SELECT` with `:name` placeholders.

```json
{
  "schema_version": "1",
  "id": "pending_posts",
  "name": "Pending posts",
  "sql": "SELECT id, title, status, created_at FROM posts WHERE status = :status ORDER BY created_at DESC",
  "params": { "status": { "type": "string", "default": "pending" } }
}
```

Bind it from a view (for example a `Table` `rows` prop):

```json
"rows": {
  "bind": { "source": "database.query", "query_id": "pending_posts", "params": { "status": "pending" } }
}
```

## `workflow_ref`

Always the workflow UUID returned by `workflows_create` / `workflows_list` — never a slug, key or name.
