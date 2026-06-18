# Changelog content

Release notes live in this folder as **one file per release**. The Portico renderer discovers them automatically — you do not add changelog pages to `docs.json` navigation.

## File naming

Use an ISO date slug:

```
changelog/2026-05-16.mdx
```

Published at `/changelog/2026-05-16`.

## Frontmatter

| Field | Required | Description |
|-------|----------|-------------|
| `title` | Yes | Release headline |
| `date` | Yes | ISO date (`YYYY-MM-DD`) — should match the filename |
| `version` | No | Semver tag shown on the timeline |
| `areas` | No | Filter tags (`workflows`, `api`, `platform`, …) |
| `summary` | No | One-line card blurb on the index |
| `added` / `changed` / `deprecated` / `removed` / `fixed` / `security` | No | [Keep a Changelog](https://keepachangelog.com/) sections (arrays) |

Structured sections are **optional**. You can instead write freeform markdown below the frontmatter, or mix both.

## Example (structured)

```yaml
---
title: Workflow Studio GA
date: 2026-05-16
version: "2.4.0"
areas: [workflows, integrations]
summary: Workflow Studio GA and new MCP connectors.
added:
  - "Visual workflow editor"
fixed:
  - "Webhook duplicate triggers"
---
```

## Example (markdown sections)

```md
---
title: Platform patch
date: 2026-01-10
---

## Added
- New feature

## Fixed
- Bug fix
```

## RSS

Each release is included in `/changelog/rss.xml` with its own permalink.

Copy `_template.mdx` to start a new entry.
