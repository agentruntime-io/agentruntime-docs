# Changelog content

User-facing release notes — one file per release (or rollup). Portico discovers them automatically; do not add pages to `docs.json`.

## Files

| File | Version | Scope |
|------|---------|--------|
| `2026-04-30.mdx` | v0.1.0 | Billing UI, Stripe checkout foundations, API gateway hardening |
| `2026-05-22.mdx` | v0.1.1 | Onboarding, Autopilot, Studio dry-run preview, webhooks, analytics, Lua steps |
| `2026-05-25.mdx` | v0.1.2 | for_each, memory kernel, chat BFF wiring, billing profiles, run summaries |
| `2026-06-05.mdx` | v0.1.3 | Google connections, password reset, HITL realtime, Workspace Chat alpha |
| `2026-06-15.mdx` | v0.1.4 / v0.1.5 | Operate hub, Studio dry-run GA, triggers, vault, multi-product billing |
| `2026-06-20.mdx` | v0.2.0 | UE baseline — session workflow, Continue on limit, in-process agents, Work/contacts |
| `2026-06-23.mdx` | v0.2.1 | Message queue, drafts, archive, Cedar permissions, lifecycle accountability |
| `2026-07-04.mdx` | v0.2.2 | Production tag alignment, workflow template panic fix, billing rate limits, chat payload patches |
| `2026-07-05.mdx` | v0.2.3 | Execution Timeline tab, composite inner step lifecycle events |
| `2026-07-22.mdx` | v0.3.0 | MCP validation, workflow pins, OAuth2 discovery, team management, billing rollup |
| `2026-08-03.mdx` | v0.3.1 | MCP-to-LLM depends_on fix, LLM step timeouts, Workspace Chat offline drafts, Google connect |
| `_template.mdx` | — | Copy to start the next entry |

**Source of truth:** GitHub release bodies (`gh release view`), cross-checked with git tags.

## File naming

ISO date slug → `/changelog/YYYY-MM-DD`. Use a suffix only when a second page shares the same date.

Copy `_template.mdx` to start a new entry.

## RSS

Each file is included in `/changelog/rss.xml` with its own permalink.
