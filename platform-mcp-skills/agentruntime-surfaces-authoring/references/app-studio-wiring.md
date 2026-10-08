# App Studio wiring (MCP agents)

**Problem:** Saving view JSON and query JSON is **not** the same as a **working app**. App Studio shows “attached workflows” and browse **data sources** from what you persist and what **compile** accepts.

There is **no** Platform MCP tool named “attach workflow” or “attach data source”. Wiring is **declarative in JSON** plus a few dedicated PUT routes.

## What “attachment” means in AgentRuntime

| Studio concept | What actually persists | MCP / API |
|----------------|------------------------|-----------|
| **Attached workflows** (view header) | Top-level `workflows[]` on the **view document** **and** every `workflow_ref` inside `bind` / `events.*.action` | `surfaces_apps_put_view` — compile upserts `surface_app_workflow_usage` |
| **Run / approve actions** | Button `events.onClick.action` with `type`: `workflow.run`, `workflow.operation`, or `workflow.cancel` | Same PUT view body; runtime executes via **`POST /v1/surfaces/action`** (no Platform MCP wrapper today) |
| **Data source (`database.query`)** | Row in `surface_app_queries` + `bind.source: "database.query"` with matching `query_id` | `surfaces_apps_put_query` **before** view binds reference the id |
| **HITL gate → view route** | `surface_app_gate_bindings` (separate from view tree) | `surfaces_apps_put_gate_bindings` |

Console parity: monorepo `docs/surfaces/CONSOLE_GUIDE.md` (Preview tab starts runs; Queries tab defines SQL).

## Order of operations (agents)

```text
1. Publish workflows → real workflow UUIDs (workflows_publish_version).
2. surfaces_apps_create → app_id (use as app_instance_id in render/action APIs).
3. surfaces_apps_put_query for each query_id used in tables/lists.
4. surfaces_apps_put_view for each view_id in navigation.
   - Include view.workflows[] for Studio’s attached list.
   - Include input fields + start button (see below), not only display binds.
5. surfaces_apps_put_gate_bindings when human_task + dedicated review view.
6. Functional verification (below) before claiming “app works”.
```

## Workflow inputs (capture text, required trigger fields)

Clicking **Start** does **not** open a separate platform modal. Required workflow inputs must be **on the view** (or fixed in the action). They become the workflow **start `params`** / trigger payload when the button fires.

1. Read required keys from the published workflow — **`workflows_graph_schema`** / Studio input schema (`input` / `run_setup.trigger_payload` defaults). See [workflow-authoring deployment-inputs.md](../../agentruntime-workflow-authoring/references/deployment-inputs.md).

2. **Pattern A — form on the view (Console “Collect form inputs”)**

   - Add inputs (`TextArea`, `TextField`, …) with `props.workflow_input: { "param": "<input_key>" }` and `editable: true`.
   - Start button:

```json
"events": {
  "onClick": {
    "action": {
      "type": "workflow.run",
      "workflow_ref": "<published-ingestion-workflow-uuid>",
      "collect_workflow_inputs": true
    }
  }
}
```

   At click, runtime walks the view tree, reads each `workflow_input.param` from component `value` / `checked` (or action `draft`), and merges into start params. Studio: **Import workflow fields** on the canvas does this wiring.

3. **Pattern B — fixed literals in the action** — use when all required inputs are constants:

```json
"params": {
  "source_id": "demo-001",
  "capture_text": "Known test paragraph for E2E."
}
```

   For row-driven starts, use `params` with `{ "from": "selection", "path": "id" }` per `SURFACES_SCHEMA.md` §6.

4. **Pattern C — MCP / API functional test (recommended evidence)**

   Platform MCP has **`workflow_run`** (not `surfaces/action`). Prove the ingestion workflow accepts data with an explicit **`trigger_payload`** matching the graph’s required inputs — then use returned **`run_id`** in render `context` for run-mode UI checks.

**Insufficient:** `params: {}` on the button when the workflow requires `capture_text`, `mailbox`, etc., and no `workflow_input` fields on the view.

## Capture / ingest views (common CRM mistake)

**Insufficient:** Text/Label components bound to `workflow.node.output` or static instructions only.

**Required for operator launch:**

1. View `workflows[]` includes the ingestion workflow UUID (`role: "primary"` is typical).
2. Input fields + **`workflow.run`** button (Patterns A or B above).
3. Optional **review** path: `human_task` step + **`workflow.operation`** approve/reject buttons **only if** the product includes review gates — see gate section below.

Reference: monorepo `docs/surfaces/examples/tiktok/publish-view.json` (gate actions) + `SURFACES_SCHEMA.md` §6.

## Browse / list views (`database.query`)

**Insufficient:** `surfaces_apps_put_view` with `database.query` bind but **no** query document, or SQL that does not match Tenant Data tables.

**Required:**

1. `surfaces_apps_put_query` with the same `query_id` as in the bind.
2. SQL is **read-only SELECT** scoped to provisioned tables.
3. **`params` on the bind** (in the view document) supply query parameters — not the render API:

```json
"bind": {
  "source": "database.query",
  "query_id": "people_list",
  "params": { "status": "active" }
}
```

4. View `workflows[]` may use `role: "data"` when browse-only (see `queue-view.json`).

## Compile status (`bind_compile`)

Every `surfaces_apps_put_view` runs compile and stores `bind_compile` on the view row.

| `bind_compile.status` | Expected meaning |
|-----------------------|------------------|
| `ok` | Workflow refs are UUIDs; bound `node_id` values exist on **active published** graph |
| `import_pending` | Non-UUID `workflow_ref` on a **workflow** binding or action |
| `error` | Unknown workflow, missing active version, or bad `node_id` |

**Spec:** `database.query` binds should **not** set `import_pending` (no `workflow_ref`). If compile reports `import_pending` on a pure `database.query` path, treat that as **unexpected platform behaviour** — capture `binding_path`, view id, and `bind_compile` JSON and report internally; do not assume the agent mis-authored the bind.

**Agents must read back:** `surfaces_apps_get_view` → `bind_compile`; `surfaces_apps_get` → app `bind_compile_status`.

## `surfaces_render` (exact MCP request)

The tool accepts **`app_instance_id`**, **`view_id`**, optional **`host`**, optional **`context`**, optional **`fixtures`**. It does **not** accept `mode` or query `params` — those are **response** `mode` (derived) and **view bind** `params` (authored).

**Browse list (no run yet)** — empty context; inspect response:

```json
{
  "app_instance_id": "<surface app uuid from surfaces_apps_create>",
  "view_id": "people",
  "host": "preview",
  "context": {}
}
```

Expect **`mode": "browse"`** in the **response** when the view uses `database.query` / `connection.read` and `context.run_id` is absent.

**Start shell (CTA present, no run)** — same request on a capture view with a `workflow.run` button; expect **`mode": "start"`** in the response.

**Run-bound view (after a real start)** — pass run context:

```json
{
  "app_instance_id": "<app uuid>",
  "view_id": "capture",
  "host": "preview",
  "context": {
    "run_id": "<from workflow_run or surfaces/action>",
    "workflow_ref": "<published workflow uuid>"
  }
}
```

Expect **`mode": "run"`** in the response, plus optional `run`, `gate`, `subscription`. Read **`interaction.mode`** — do not treat `tree` alone as proof of a live run.

Optional **`fixtures`** (preview host): mock binding values for designer-only previews — not a substitute for live Tenant Data rows.

Human-task inbox render may also set `context.task_id` (see monorepo `examples/tiktok/render-request.json`).

## Functional evidence (required before “app works”)

Structural checks (button present, compile ok, render returns a tree) are **necessary but not sufficient**.

| Claim | Required evidence |
|-------|-------------------|
| **Ingestion / start works** | **`workflow_run`** (or Console Preview / `POST /v1/surfaces/action`) with a **known `trigger_payload`** → returned **`run_id`**; optional **`workflow_await`** / **`runs_get_summary`** showing expected step progress. JSON containing `workflow.run` is not proof. |
| **Browse read works** | Seed a **known row** (Tenant Data insert or workflow write) with a stable id you control → **`surfaces_render`** (browse context) → **`tree`** (or follow-up tenant query) shows that id/value. Empty table / placeholder labels are not proof. |
| **Review / approve works** | **Only when** the app includes **`human_task`** gates: run reaches gate → **`workflow.operation`** or **`tasks_complete`** with **`context.run_id`** + **`task_id`** → gate clears or expected state change. Skip this row if there is no review gate. |
| **Gate routing** | If using inbox/review views: **`surfaces_apps_list_gate_bindings`** matches `(workflow_ref, node_id, view_id)`. |

List **`checks_performed`** with app id, view ids, compile statuses, **`run_id`**, and the **known test id** used for browse proof.

## Product gaps (do not pretend otherwise)

- No MCP **`surfaces_action`** — button parity in automation may require BFF **`POST /v1/surfaces/action`** or operator Console until a tool exists.
- No MCP “attach workflow to app” besides view JSON + compile side effects.
- Compile runs on view PUT only; occasional **`import_pending`** on non-workflow binds should be reported as platform bugs.

When Studio and MCP disagree, monorepo **`docs/surfaces/SURFACES_RUNTIME_CONTRACTS.md`** is authoritative for compile and render behaviour.
