---
name: backlog-triage
description: >
  Beton Kemon backlog triage. Processes ideas into tasks/open/ with
  proper frontmatter, debates non-trivial items before adding,
  recommends what to pick up next, and grooms the open queue. Replaces
  the older /todo skill's category-based flow now that backlog lives in
  tasks/open/. Use when asked to "triage", "add to backlog", "what's
  next", "groom tasks", or generally manage tickets.
---

# /backlog-triage - Beton Kemon Backlog Manager

Backlog source of truth: `tasks/open/`. One markdown file per ticket. Format and conventions: `tasks/README.md`.

## Modes

Pick from user input. Default to **add** if ambiguous.

### `add` - debate, then triage

**Do not silently add.** The user wants pushback, not a stenographer.

For each idea:

1. **Restate** in one line so the user knows you got it.
2. **Debate** - 2-4 pros, 2-4 cons, specific to *this* codebase: cold-start phase, mobile-first, >=5 hardgate (ADR-0004), no PII (ADR-0003), current Wave priorities in `tasks/open/001`-`004`.
3. **Overlap check** - name any existing ticket this competes with, duplicates, or depends on. (Use `ls tasks/open/` and grep titles.)
4. **Recommend** - *add as-is*, *add with modification*, *defer until X*, or *drop*. Pick one.
5. **Wait** for user approval.

Multiple ideas at once: debate each separately.

Trivially good and small (typo fix, obvious copy tweak): skip the debate, confirm placement, add.

Once approved:

- **Assign next id**: `ls tasks/open/ tasks/done/ | grep -oE '^[0-9]+' | sort -n | tail -1`, increment.
- **Filename**: `tasks/open/NNN-kebab-slug.md`.
- **Frontmatter** per `tasks/README.md`:
  ```yaml
  ---
  id: NNN
  title: Verb-led title (<=12 words)
  area: admin | public-ui | data | search | trust | i18n | testing | infra | design | ops
  priority: existential | high | medium | low
  status: open
  created: 2026-MM-DD
  ---
  ```
- **Body** sections: ## Why, ## Acceptance, optionally ## Open questions, ## Related.

### `next` - recommend what to pick up

Read `tasks/open/` (especially low NNN; they're prioritized first). Read `AGENTS.md` for current state. Suggest 1-3 candidates with one-line rationale. Bias toward:

- `priority: existential` items not yet started.
- Items that unblock cold-start / contributor push.
- Small, shippable items over sprawling ones.
- Bugs over features when both exist.

Don't start work. Wait for the user to pick.

### `start <id-or-slug>` - flip status to in-progress

Edit the ticket frontmatter: `status: open` -> `status: in-progress`. Keep in-progress to 1-2 items max; if already at 2, ask before adding a third.

### `done <id-or-slug>` - close ticket

```bash
git mv tasks/open/NNN-slug.md tasks/done/NNN-slug.md
```

Update frontmatter: `status: done`, add `closed: YYYY-MM-DD`. Optionally append a `## Resolution` section with commit SHA(s).

Per the user's commit-per-todo memory, the user commits per completed task - if there's no commit yet for the work, remind them.

### `status` - quick read-out

Counts by priority + area. List in-progress tickets. List 3 most recently-closed. <15 lines.

```bash
# Useful one-liners:
ls tasks/open/ | wc -l
grep -l 'priority: existential' tasks/open/*.md
grep -l 'status: in-progress' tasks/open/*.md
ls -t tasks/done/ | head -3
```

### `groom` - tidy the queue

- Look for stale `in-progress` (status set, no recent commits to related files). Ask the user.
- Look for tickets with `blocked-by: human` and check if the human action has happened (`docs/HUMAN-INTERVENTIONS.md`).
- Flag tickets with vague acceptance criteria - propose tightening.

## Scope guardrails (cross-check against AGENTS.md hard rules)

If an incoming idea would violate one of these, call it out and ask before adding:

- No emoji in production UI (rule 1)
- No em dash in user-facing text (rule 2, ADR-0001)
- No PII on `salary_entries` (rule 3, ADR-0003)
- >=5 hardgate must hold (rule 4, ADR-0004)
- Idempotency + transactional aggregate recompute (rule 5)
- `revalidatePath("/admin/queue")` synchronous (rule 6, ADR-0005)
- DB pool config load-bearing (rule 7, ADR-0006)
- Single canonical English data form (rule 9, ADR-0007)

## Tone

Terse. After any edit: 2-6 lines summary, then stop.

## Gotchas

- **Never `done` a ticket without dev verification.** "Typecheck passes + commit landed" is not verification per AGENTS.md - the user has pushed back hard ("Did we even verify using dev if this actually works? Why would you close a ticket before that?"). Browser-verify each acceptance criterion via Playwright before `git mv` into `tasks/done/`.
- **Don't flip status to `in-progress` without a stub commit.** A ticket sitting in-progress with no commits against the related files becomes ambiguous during `groom` - did work start and stall, or did the status drift? Commit at least a stub so the audit trail is real.
- **When in doubt, route exploratory asks to `/ideate` first.** If the user is brainstorming ("what if we...", "I have an idea") rather than instructing, debating an idea before triaging it into `tasks/open/` is the right shape. Filing a ticket too eagerly creates backlog noise.
- **Cross-check incoming ideas against the hard rules (1-13) in AGENTS.md.** Tickets that would violate "no emoji in UI", "no em-dash in user-facing text", "no PII on `salary_entries`", "pool max load-bearing", etc. need to be flagged at intake, not after someone starts work.
- **Bangla copy in tickets goes in proper Bangla script.** When a ticket includes user-facing strings, write them in proper Bangla in the markdown body (mirror what `messages/bn.json` will see). Use Banglish only when discussing the string in chat.

## Related

- `tasks/README.md` - format and workflow
- `CONTRIBUTING.md` - branch + commit conventions
- `AGENTS.md` - hard rules

## Ask at forks

If the work hits a real decision (architecture, scope, ticket-shape, user-facing copy/UX), use `AskUserQuestion` with concrete options + a recommendation - even in auto mode. See AGENTS.md "Working norms".

## Self-update on better patterns

If this session surfaces a better way of doing things (a new fork to flag, a tool that should have been preferred, a recurring confusion worth a paragraph), propose the edit to this skill / `AGENTS.md` / `docs/*.md` *in the same session*. Do not stash in per-agent memory. See AGENTS.md "Working norms".
