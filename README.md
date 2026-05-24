# Builder's Playbook — reference files

These are the open-source AI-harness files referenced in the appendix of
**The Builder's Playbook** (PDF, by Ishmam Chowdhury).

Each file lives at the same path it has in the production [Beton Kemon](https://www.betonkemon.com)
repository (which is itself private). They are mirrored here so the playbook's
URLs resolve and any reader can copy what they need:

| Path | What it is |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | The contract file the AI agent reads first every session. Same content as `AGENTS.md`. |
| [`AGENTS.md`](AGENTS.md) | Same content as `CLAUDE.md`. Both filenames exist so either convention resolves. |
| [`docs/sdlc.md`](docs/sdlc.md) | How a single ticket moves from open to done. |
| [`docs/product/contribute.md`](docs/product/contribute.md) | One per-feature doc, written for the agent to read before touching the contribute surface. |
| [`.claude/skills/debug/SKILL.md`](.claude/skills/debug/SKILL.md) | A real skill — what the agent loads automatically when you say `/debug`. |
| [`design/redesign-prompt.md`](design/redesign-prompt.md) | The verbatim prompt fed into Claude Design to generate the brand's design system. |

## How to use

Copy what you want. Replace the project-specific lines with yours. The whole
point of the playbook is that you should not have to invent these from scratch
on day zero of your own build.

## How this repo stays in sync

This is a mirror, not a primary. The source files live in the private Beton Kemon
repo. They are pushed here by `scripts/sync-companion.mjs` whenever a referenced
file changes.

---

[github.com/Ishmam156](https://github.com/Ishmam156) · [linkedin.com/in/ishmamchowdhury](https://www.linkedin.com/in/ishmamchowdhury) · [betonkemon.com](https://www.betonkemon.com)
