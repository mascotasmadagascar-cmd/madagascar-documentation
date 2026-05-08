# Contributing to This Documentation

## Who Should Edit This

- **Project owner / lead developer** — architecture decisions, cost updates, new service integrations
- **Staff onboarding** — add clarifications, correct outdated steps
- **AI assistants (Claude, Codex)** — when instructed to update docs alongside code changes

## What Lives Here vs in the Codebase

| Here (this repo) | In the codebase |
|------------------|----------------|
| Architecture decisions and rationale | Inline code comments |
| Service limits and free tier analysis | `RULES.md` files per project |
| Deployment procedures | `package.json` scripts |
| Secret names and rotation policy | `.dev.vars.example` |
| Business context and compliance notes | — |

`RULES.md` files in each project root (`cf-astro/RULES.md`, `cf-admin/RULESAd.md`, etc.) are the **single source of truth** for hard constraints — free tier limits, banned patterns, and architectural invariants. Keep this documentation consistent with them.

## How to Edit

1. Edit the relevant `.md` file directly — no build step required
2. For new services or major architecture changes, update:
   - The relevant project `README.md`
   - `01-architecture-overview.md` (service map)
   - `12-service-connections.md` (dependency map)
   - `13-free-tier-master-reference.md` (if new free tier limits apply)
3. For cost changes, update `03-pricing-and-cost-analysis.md` and the project's cost doc
4. Keep the root `README.md` "Key Numbers" table current

## Markdown Conventions

- **Mermaid diagrams**: use fenced ` ```mermaid ` blocks — GitHub renders them natively
- **Tables**: always include a header row and alignment row
- **File links**: always use relative paths (`./cf-admin/authentication.md`) — never absolute URLs
- **Code blocks**: always include the language identifier (` ```typescript `, ` ```sql `, ` ```bash `)
- **No emojis** in technical sections — use them sparingly in headings if needed for navigation

## Blockers and Active Work

Known gaps and in-progress items are tracked in [Implementations & Blockers](./implementations-and-blockers.md). Add new blockers there rather than leaving TODOs scattered in individual docs.
