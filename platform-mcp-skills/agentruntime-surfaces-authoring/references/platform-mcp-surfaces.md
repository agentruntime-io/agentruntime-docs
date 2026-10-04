# Platform MCP — Surfaces apps

There is **no** separate attach-workflow or attach-data-source tool. Wiring is **`surfaces_apps_put_view`** (workflows, binds, actions) + **`surfaces_apps_put_query`** (browse SQL) + **`surfaces_apps_put_gate_bindings`** (HITL). See skill [app-studio-wiring.md](../agentruntime-surfaces-authoring/references/app-studio-wiring.md).

| Tool | Method | Purpose |
|------|--------|---------|
| `surfaces_schema_catalog` | — | Schema URLs |
| `surfaces_apps_list` | GET | List apps |
| `surfaces_apps_create` | POST | New app shell |
| `surfaces_apps_get` | GET | Read app |
| `surfaces_apps_update` | PATCH | Patch manifest/navigation |
| `surfaces_apps_put_view` | PUT | Upsert view document |
| `surfaces_apps_get_view` | GET | Read view |
| `surfaces_apps_put_query` | PUT | Upsert database.query |
| `surfaces_apps_get_query` | GET | Read query |
| `surfaces_apps_publish_package` | POST | Catalog app package |
| `surfaces_render` | POST | Preview render |
| `surfaces_apps_list_gate_bindings` | GET | HITL gate map |
| `surfaces_apps_put_gate_bindings` | PUT | HITL gate map |

`app_id` is the surface app UUID from create/list.
