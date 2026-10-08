---
name: agentruntime-platform-design
description: End-to-end external-agent playbook for Tenant Data + workflows + Surfaces apps. Orchestrates the three authoring skills and Platform MCP apply/validate tools.
---

Use this when an external agent should **design and deploy** a solution (CRM, inbox, extract-to-db) without Console clicking.

## Hosted copy

**Browse:** https://github.com/agentruntime-io/agentruntime-docs/tree/main/platform-mcp-skills/agentruntime-platform-design

**Fetch:** https://raw.githubusercontent.com/agentruntime-io/agentruntime-docs/main/platform-mcp-skills/agentruntime-platform-design/SKILL.md

## Bootstrap

1. **`ar_get_started`** — tenant, capabilities, `platform_mcp.publishable_tool_count`, skill URLs, **`recommended_next_actions`** (playbook **`design_solution`**).
2. If a raw URL fails, use `ar_get_started` → `documentation.embedded_schema_tools` (MCP tools return schemas/contracts inline).
3. Load sub-skills (order matters):
   - [agentruntime-tenant-data-authoring](../agentruntime-tenant-data-authoring/SKILL.md)
   - [agentruntime-workflow-authoring](../agentruntime-workflow-authoring/SKILL.md)
   - [agentruntime-surfaces-authoring](../agentruntime-surfaces-authoring/SKILL.md)

## Phased delivery (CRM-style)

See [references/phased-crm-pattern.md](references/phased-crm-pattern.md) — v1 ingest→review, v2 parse LLM JSON, v3 apply, v4 Surfaces. Do not claim a full CRM after v1.

## Recommended pipeline

```text
Phase A — Data:
  tenantdata_logical_schema + tenantdata_idempotency_patterns
  → tenantdata_databases_create OR migrate
  → tenantdata_api_contract_get → tenantdata_mcp_provision

Phase B — Workflows:
  workflows_authoring_contract → workflows_graph_schema → workflows_llm_step_contract (if llm_call)
  → workflows_create → workflows_update
  → workflows_validate (+ tenantdata_schema) → workflows_publish_version

Phase C — UI (optional):
  surfaces_schema_catalog → surfaces_apps_create
  → surfaces_apps_put_query (browse tables first)
  → surfaces_apps_put_view (+ view.workflows[], workflow.run / workflow.operation actions)
  → surfaces_apps_put_gate_bindings (HITL)
  → verify bind_compile + functional evidence (app-studio-wiring.md — not render `mode` in request)

Phase D — Blobs (optional):
  files_upload_content → files_mcp_provision

→ workflow_run (after validate_before_run playbook)
```

## Storage decisions

Read tenant-data **`storage-stack.md`** before column design:

- Rows → Tenant Data
- Blobs → Work files + reference column
- UI metadata → Surfaces JSONB
- Secrets → Vault / connections

## Evidence bar

Do not claim “deployed” without:

- `api_contract` version after schema apply
- `mcp_instance_id` from provision
- `workflows_validate` valid (+ run id if executed)
- **Surfaces (if built):** per **`app-studio-wiring.md`** — functional start (`workflow_run` + `run_id`) and browse proof with a known row; `bind_compile` ok; approve evidence only if the app has review gates.

Cross-skill reviews: issue table with artifact path, rule, fix.

