# AgentRuntime documentation

User-facing docs for [docs.agentruntime.io](https://docs.agentruntime.io), built on [Mintlify](https://mintlify.com).

## About this project

- MDX pages with YAML frontmatter
- Site config in `docs.json`
- Preview locally: `mint dev`
- Check links: `mint broken-links`

## Terminology

Use AgentRuntime product terms consistently:

| Term | Usage |
|------|-------|
| **AgentRuntime** | Product name (one word, capital R). Not "Agent Runtime". |
| **Console** | Web UI at console.agentruntime.io |
| **Workspace / tenant** | Top-level org container; prefer **workspace** in user-facing prose |
| **Project** | Unit that scopes workflows, runs, MCP instances |
| **Workflow Studio** | Visual workflow editor |
| **Command Center** | Human task and run inbox |
| **MCP instance** | Installed tool server in a project |
| **Connection** | Saved integration credentials |
| **Run** | Single workflow execution |
| **Autopilot** | Console copilot in Chat |
| **PAT** | Personal access token for API access |

### Roles

Use exact role slugs when documenting permissions:

- `tenant_admin`
- `project_admin`
- `project_contributor`
- `project_viewer`

Do not use generic Admin/Member/Viewer from the Mintlify template.

## Style preferences

- Active voice, second person ("you")
- Sentence case for headings
- Bold for UI elements: Click **Workflow Studio**
- Code formatting for paths, commands, API paths, and adapter slugs
- Link to the real Console (`https://console.agentruntime.io`) and API (`https://api.agentruntime.io`)

## Content boundaries

**Document:**

- Console workflows, integrations, billing, and API usage
- Real connector catalog (Go adapters in `connectors/GO_CONNECTORS.md`)
- Actual auth methods (email, Google OAuth, PATs)

**Do not document:**

- Internal service URLs (control-service, agentruntime engine direct)
- Sysadmin operator tools
- Features behind prod feature flags unless marked as preview/beta

When adding connector docs, verify the adapter exists in `connectors/GO_CONNECTORS.md` before listing it.

## Localized content (portal)

The first-party docs portal (`doc_system/portal`) serves English at unprefixed URLs (`/platform/quickstart`) and translations under locale prefixes (`/es/platform/quickstart`).

Place translated MDX beside the English tree using this convention:

```
i18n/
  es/
    index.mdx
    introduction/what-is-agentruntime.mdx
    platform/quickstart.mdx
  fr/
    ...
```

- Mirror the English page path: `platform/quickstart.mdx` → `i18n/es/platform/quickstart.mdx`
- Keep the same frontmatter keys (`title`, `description`, `sidebarTitle`)
- If a translation is missing, the portal falls back to English and shows a banner
- Mintlify preview (`mint dev`) still reads root-level English MDX only

## Doc structure

```
introduction/     Product overview
platform/           Workspace setup, settings, members, roles, billing, analytics, security, referrals, feature availability, troubleshooting
workflows/          Studio, Command Center, step types, runs, patterns, human tasks, JSON cookbook
integrations/       Integration setup (connections, MCP, custom MCP, auth, webhooks, webhook wizard, catalog index)
connectors/         One MDX per MCP adapter slug (40 Go connectors)
guides/             End-to-end multi-connector workflow recipes (index + 5 tutorials)
ai/                 LLM providers, Autopilot, chat, memory (mark preview if Console UI flag off)
api/                REST overview, authentication, examples, reference, Platform MCP
```

## Connector guides

Published pages live in `connectors/{adapter-slug}.mdx` and are **hand-maintained**.

When adding or updating tools in code:

1. From `connectors/go-connectors/`, generate a **staging draft** (gitignored):

   ```bash
   go run ./cmd/connector-doc -connector linkedin-connector -format mdx
   go run ./cmd/connector-doc -all -format mdx
   ```

   Output: `agentruntime-docs/.generated/connectors/{slug}.mdx`

2. Copy tools, config keys, and field tables from the draft into the published page — rewrite setup/auth prose for users.
3. Never point Mintlify at `.generated/` — it is reference only.
4. Add new slugs to `integrations/connector-catalog.mdx` and `docs.json` Connectors group.
5. Keep **integration setup** docs free of per-connector tool tables — link to the connector page instead.

### Connector page template

Every published `connectors/{slug}.mdx` page uses the **same section order** so pages feel consistent in the sidebar and on screen. Do not skip sections — use a short `<Note>` when a section does not apply (e.g. no config keys).

| Section | Purpose |
|---------|---------|
| Frontmatter `title` / `description` | `title`: `{Name} connector`. `description`: one-line value prop, not "MCP connector — tool X". |
| Intro (no heading) | What the service is and why automate it in AgentRuntime (2–3 sentences). |
| `## Prerequisites` | Accounts, API access, roles, network — bullets only. |
| `## Connect in AgentRuntime` | `<Steps>` using real Console labels: **Connections** (`New custom connection`, **Google account** card, **Connect WhatsApp**), **MCP** (`Add instance`), **Instance config** (wire connection, **active** profile). Smoke-test with **mcp_call** — not "Validate bindings". |
| `## What you can build` | 3–4 **concrete workflow ideas** with bold titles and one sentence each. Tie to real tools. |
| `## Tools` | Table from staging draft or connector README. Group with `###` subheadings if more than ~15 tools. |
| `## Example` | One `mcp_call` JSON block showing a realistic step. |
| `## Configuration` | Config keys table when the adapter has required settings beyond the connection. |
| `## Troubleshooting` | Small table (2–4 rows). Omit only if nothing useful to say. |
| `## Related` | Always link: [Integrations quickstart](/integrations/quickstart), [Connector catalog](/integrations/connector-catalog), [Troubleshooting](/platform/troubleshooting). Add one contextual link (e.g. [Google Workspace](/integrations/google-workspace) for Google adapters). |

**Google Workspace adapters** (`gmail`, `google-*`): prerequisites must reference [Google Workspace setup](/integrations/google-workspace). Connection steps are shorter — OAuth is shared; each page focuses on **use cases** for that product.

**Stubs** (`slack` with no tools yet): keep the same structure; add `<Warning>` or `<Note>` under Tools explaining preview/limited status.

Never copy staging draft prose verbatim — rewrite for users. Staging lives in `.generated/connectors/` only.

### End-to-end guides

Full multi-connector recipes live in `guides/`. Each guide includes:

- Outcome diagram (mermaid), prerequisites, connectors table
- Console build steps and copy-paste step JSON
- Variations and links to connector pages

Add new guides to `docs.json` **Guides** group and link from `guides/overview.mdx` and `integrations/connector-catalog.mdx` **Common patterns** table.
