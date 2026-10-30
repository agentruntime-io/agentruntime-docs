# Changelog content

User-facing release notes — one file per release (or rollup). Portico discovers them automatically; do not add pages to `docs.json`.

## Files

| File | Version | Scope |
|------|---------|--------|
| `2026-04-30.mdx` | v0.1.0 | Billing UI, Stripe checkout foundations, public API hardening |
| `2026-05-22.mdx` | v0.1.1 | Onboarding, Autopilot, Studio dry-run preview, webhooks, analytics, Lua steps |
| `2026-05-25.mdx` | v0.1.2 | for_each, workflow memory, Workspace Chat foundations, billing profiles, run summaries |
| `2026-06-05.mdx` | v0.1.3 | Google connections, password reset, HITL realtime, Workspace Chat alpha |
| `2026-06-15.mdx` | v0.1.4 / v0.1.5 | Operate hub, Studio dry-run GA, triggers, vault, multi-product billing |
| `2026-06-20.mdx` | v0.2.0 | Conversation workflows, Continue on limit, faster agent_call steps, Work/contacts |
| `2026-06-23.mdx` | v0.2.1 | Message queue, drafts, archive, stronger API permissions, lifecycle accountability |
| `2026-07-04.mdx` | v0.2.2 | Empty-template run fix, billing rate limits, chat payload patches, connector authorization |
| `2026-07-05.mdx` | v0.2.3 | Execution Timeline tab, composite inner step events |
| `2026-07-22.mdx` | v0.3.0 | MCP validation, workflow pins, OAuth2 discovery, team management, billing rollup |
| `2026-08-03.mdx` | v0.3.1 | MCP-to-LLM depends_on fix, LLM step timeouts, Workspace Chat offline drafts, Google connect |
| `2026-09-19.mdx` | v0.3.2 | for_each fan-out (inner workflow_call), iteration budgets, loop-body Sub-workflow drag, child approval polling |
| `2026-09-29.mdx` | v0.10.4 | workflow_call parent input, skip_when, shared inbound connections, trial billing UX, work file base64 upload |
| `2026-09-30.mdx` | v0.10.5 | Autopilot conversation flow (Ask vs Autopilot), graph dry-run, Console actions from chat |
| `2026-10-05.mdx` | v0.12.0 | Mid-turn messages, graph dry-run UX, Platform MCP skills |
| `2026-10-08.mdx` | v0.13.0 | Sub-workflow step reliability (v0.14.0 on 2026-10-14 had no customer-facing changes) |
| `2026-10-29.mdx` | — (per-service releases, no platform tag) | Channel files, session choices, Missed tab, package updates, member hand-over |
| `_template.mdx` | — | Copy to start the next entry |

**Source of truth:** GitHub release bodies (`gh release view`), cross-checked with git tags.

## File naming

ISO date slug → `/changelog/YYYY-MM-DD`. Use a suffix only when a second page shares the same date.

Copy `_template.mdx` to start a new entry.

## RSS

Each file is included in `/changelog/rss.xml` with its own permalink.
