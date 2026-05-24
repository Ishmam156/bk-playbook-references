# Builder's Playbook — reference files

Open-source files referenced in the appendix of **The Builder's Playbook** (PDF, by Ishmam Chowdhury).

Each file lives at the same path it has in the production [Beton Kemon](https://www.betonkemon.com) repository (which is itself private). They are mirrored here so the playbook's URLs resolve and any reader can copy what they need.

## Appendix 1 — the AI-harness layer

| Path | What it is |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | The contract file the AI agent reads first every session. Same content as `AGENTS.md`. |
| [`AGENTS.md`](AGENTS.md) | Same content as `CLAUDE.md`. Both filenames exist so either convention resolves. |
| [`docs/sdlc.md`](docs/sdlc.md) | How a single ticket moves from open to done. |
| [`docs/product/contribute.md`](docs/product/contribute.md) | A per-feature doc, written for the agent to read before touching the contribute surface. |
| [`design/redesign-prompt.md`](design/redesign-prompt.md) | The verbatim prompt fed into Claude Design to generate the brand's design system. |

## Appendix 2 — skills you can fork

A *skill* is a small markdown file under `.claude/skills/<name>/SKILL.md` that the agent loads when you invoke it. These eight are the most-reusable ones from the project.

| Path | What it does |
|---|---|
| [`.claude/skills/ideate/SKILL.md`](.claude/skills/ideate/SKILL.md) | Pushes back on raw ideas before they become tickets. Optionally generates HTML demos. |
| [`.claude/skills/debug/SKILL.md`](.claude/skills/debug/SKILL.md) | Project-aware bug investigator for non-acute issues. No fix until a named root cause. |
| [`.claude/skills/incident/SKILL.md`](.claude/skills/incident/SKILL.md) | Production incident protocol. When the site is on fire right now. |
| [`.claude/skills/website-logs-10m/SKILL.md`](.claude/skills/website-logs-10m/SKILL.md) | Live multi-source triage (Vercel + Sentry + Supabase) over the last 10 minutes. |
| [`.claude/skills/website-logs-2h/SKILL.md`](.claude/skills/website-logs-2h/SKILL.md) | Same fanout over a 2-hour window. Trend detection and billing-impact analysis. |
| [`.claude/skills/costanalysis/SKILL.md`](.claude/skills/costanalysis/SKILL.md) | Vercel + Supabase usage live, or reconcile a billing CSV into a multi-month trend. |
| [`.claude/skills/backlog-triage/SKILL.md`](.claude/skills/backlog-triage/SKILL.md) | Processes ideas into tasks/open/ with proper frontmatter. |
| [`.claude/skills/doc-sync/SKILL.md`](.claude/skills/doc-sync/SKILL.md) | Maps changed file paths to the docs/*.md that should be updated alongside. |

## How to use

Copy what you want. Replace the project-specific lines with yours. The whole point of the playbook is that you should not have to invent these from scratch on day zero of your own build.

## How this repo stays in sync

This is a mirror, not a primary. The source files live in the private Beton Kemon repo. They are pushed here by `scripts/sync-companion.mjs` whenever a referenced file changes.

---

[github.com/Ishmam156](https://github.com/Ishmam156) · [linkedin.com/in/ishmamchowdhury](https://www.linkedin.com/in/ishmamchowdhury) · [betonkemon.com](https://www.betonkemon.com)
