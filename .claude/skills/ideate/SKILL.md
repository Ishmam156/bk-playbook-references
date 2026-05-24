---
name: ideate
description: >
  Beton Kemon idea triage. Pushes back on raw ideas before they
  become tickets, debates each one against the project's hard
  rules and current Wave priorities, optionally generates an HTML
  demo into out/ideate/ when a visual helps decide, and only files
  a ticket if the proposal earns it. Use when asked to "ideate",
  "brainstorm", "what if we...", "I have an idea", "should we
  build...", or anything that sounds like an idea seeking validation
  rather than a directive. Proactively invoke this skill (do NOT
  go straight to filing a ticket) when the user is exploring rather
  than instructing - the value is the debate, not the write.
---

# /ideate - debate before you ticket

Beton Kemon is a deliberately scoped product (`AGENTS.md`, `docs/adr/`). The default failure mode for an AI-assisted backlog is "every idea becomes a ticket" - which dilutes priority and erodes the ADR-encoded posture. **Your job here is to be a thoughtful editor, not a stenographer.**

## Mode dispatch

Most invocations are mode `propose`. Pick the one that fits the user's input:

| Mode | When |
| --- | --- |
| `propose` (default) | User shares an idea. Debate, then decide together. |
| `demo` | User wants a visual before deciding. Generate `out/ideate/<slug>.html` and pause. |
| `dossier` | Idea is interesting but ambiguous. Spawn an Explore agent to gather context, return a structured dossier. |
| `commit` | Debate is over and the user said "yes". Hand off to `/backlog-triage add` with the agreed framing. For non-trivial scope, also emit an HTML plan (see "Plan output (HTML)" below). |
| `plan` | User explicitly says "commit it", "make this a plan", "write the plan", "graduate this idea". Produce an HTML implementation plan at `out/ideate/<slug>.html` using `assets/plan-template.html`. |

If the user's input is ambiguous, restate in one line and ask which mode - don't guess.

## Mode: `propose` (the main path)

For each idea:

1. **Restate** in one line. Make sure you got it. If genuinely ambiguous, ask one short clarifying question and stop.
2. **Hard-rule check.** Does the idea brush against any of these?
   - No emoji in production UI (rule 1)
   - No em dash in user-facing text (rule 2, ADR-0001)
   - No PII on `salary_entries` (rule 3, ADR-0003)
   - >=5 hardgate must hold (rule 4, ADR-0004)
   - Idempotency + transactional aggregate recompute (rule 5)
   - `revalidatePath("/admin/queue")` synchronous (rule 6, ADR-0005)
   - DB pool config load-bearing (rule 7, ADR-0006)
   - Single canonical English data form (rule 9, ADR-0007)

   If yes, surface it explicitly. The user might still want to proceed (rules can be debated and superseded with a new ADR), but they should choose with eyes open.
3. **Debate** - specific to *this* codebase and the current state. Generic pros/cons are useless.
   - **Pros:** 2-4 bullets. What does it unlock? What user pain does it remove? Which Wave priority does it support?
   - **Cons / risks:** 2-4 bullets. What does it cost? What does it break or weaken? What existing item does it overlap with or undermine?
4. **Overlap check.** Grep `tasks/open/*.md` and `tasks/done/*.md` titles. Name any ticket this competes with, duplicates, depends on, or supersedes. Don't skip this - duplicate-ticket sprawl is a real failure mode.
5. **Architecture / scope check.** Skim the relevant `docs/product/<flow>.md` if the idea touches a user-visible flow, or `docs/architecture.md` for cross-cutting work. Note any edge case the idea would have to handle.
6. **Recommend.** One of:
   - **Kill.** Honest "this doesn't earn its weight" with one-line rationale.
   - **Refine.** Specific modification that would make it earn its weight. Re-run debate with the modified version.
   - **Defer.** "Park until X ships" with the gating dependency named.
   - **File as ticket.** Earns it; here's the proposed frontmatter.
7. **Wait.** Do not invoke `/backlog-triage` or write any file. The user must say "yes" before anything moves.

If multiple ideas at once: debate each separately. Don't merge into one recommendation.

If the idea is **trivially good and small** (typo fix, obvious copy tweak, clear bug observed): skip the full debate, recommend immediate fix or one-line ticket, ask before adding.

## Mode: `demo`

When the user wants to see what an idea would look like before deciding (especially UI changes, layout reorganizations, new patterns):

1. Generate a self-contained HTML file at `out/ideate/<slug>-<timestamp>.html`.
   - Use vanilla HTML + inline CSS that matches `--bk-*` tokens from `src/app/globals.css`. Hardcode the token values; don't import from the project (the file must open standalone).
   - Latin = Geist (link from Google Fonts CDN). Bangla = Hind Siliguri (same source).
   - No emoji, no em dash (the rules apply to demos too - if the demo violates them, the user will internalize the violation).
   - Mock data: synthesize plausible BD-shaped values. Don't query the real DB.
   - One screen per concept; if comparing variants, multiple `<section>` blocks side by side.
2. Print the file path and a one-liner on what to look for.
3. Wait for the user's verdict before recommending kill/refine/defer/file.

The demo is a thinking aid, not a deliverable. If the user wants to ship, they'll separately invoke whatever flow ships (e.g., direct Edit calls or a ticket via `commit` mode).

## Mode: `dossier`

When the idea is interesting but the user (or you) lacks the context to debate it well, spawn an Explore agent in the background to dig:

```
Agent({
  description: "Idea dossier: <slug>",
  subagent_type: "Explore",
  prompt: "..."  # specific to the idea: which files to read, which
                 # tickets to compare, which docs to check
})
```

When the agent reports back, return to mode `propose` with the new context.

## Mode: `commit`

User has said yes. Hand off to `/backlog-triage add` with the exact agreed-upon framing:

- Title (verb-led, <=12 words)
- Area (`admin | public-ui | data | search | trust | i18n | testing | infra | design | ops`)
- Priority (`existential | high | medium | low`) - default `medium` unless debate concluded otherwise
- Why (the pro-side from the debate, distilled)
- Acceptance (concrete, testable)
- Open questions (anything debate didn't resolve)
- Related (ADRs, tickets, files cited during debate)

Then stop. Do not also implement. The ticket is the artifact.

## Plan output (HTML)

When the user graduates an idea past the debate phase with phrases like "commit it", "make this a plan", "write the plan", or "graduate this idea", `/ideate` produces an **HTML implementation plan** instead of a markdown ticket body. Trivial one-liner tickets (typo fix, copy tweak, obvious bug) stay markdown via `/backlog-triage add` or `/todo`. Non-trivial tickets (anything touching schema, routes, cache, hardgate, i18n, or more than one file) get an HTML plan.

### How

1. Copy `assets/plan-template.html` to `out/ideate/<slug>.html` (the `out/ideate/` directory is gitignored, which is fine; the plan is a session input, not a repo artifact).
2. Populate every `{{PLACEHOLDER}}` from the debate transcript. Do not leave placeholders behind.
3. Fill the timestamp footer with the current time in BDT (e.g. `2026-05-12 23:30 BDT`).
4. The risk dots in the header reflect the worst-severity rows in the risk table:
   - 1 amber/red row -> `on-green on-green on-amber` or red
   - mostly mediums -> all three amber
   - any high -> at least one red dot
5. Hand the file path to the user. They will pass it as input to the next session via `/backlog-triage start`.

### Reference example

`assets/example-plan-comment-threads.html` is "this is what good looks like" - a filled plan for an admin internal-notes feature. Open it in Chrome standalone; it must render fully. Mimic its level of specificity (real file paths, real action names, real migration shape) when populating a new plan. Do not ship a plan that is vaguer than the example.

### Round-trip into `/backlog-triage start`

The HTML plan is the **handoff handshake** between the ideation session and the implementation session. The implementer runs `/backlog-triage start` and passes the HTML file path; that session:

1. Opens the file and reads every section, top to bottom.
2. Treats the milestones as the work plan, the data flow SVG as the architecture contract, the risk table as the test plan, and the open questions as **forks to resolve with the user before any edit**.
3. If any open question is still open at start time, asks the user (via `AskUserQuestion`) before touching code.
4. Files the actual ticket in `tasks/open/` only after the open questions are resolved, embedding the HTML plan's path in the ticket frontmatter under `plan:`.

### Gotchas (plan-specific)

- **An HTML plan that doesn't surface concrete Open Questions is broken.** Every plan should have at least 2 open questions because that's where the human-in-the-loop fork lives. If you wrote a plan with zero open questions, you missed forks; go back and find them.
- **The template's hard-coded `--bk-*` tokens must match `src/app/globals.css`.** When the design system updates tokens, update `assets/plan-template.html` in the same change. (Same rule as `out/ideate/*.html` demos.)
- **No em dash and no emoji inside the populated plan**, even in mockup copy or code comments that look like internal notes. The plan is read by the implementer as a contract; rule-violating content in the contract leaks into the implementation.
- **Mockups in the plan are HTML+CSS, not screenshots.** If you can't sketch the surface in 30 lines of HTML, your plan is too vague; resolve the ambiguity in the debate phase first.
- **The SVG data flow is hand-tweakable, not generated.** Edit the inline SVG to reflect the actual route -> action -> table -> cache-tag chain for this specific plan. The template's default boxes are a starting point, not the answer.

## Anti-patterns

- **Filing the ticket without the debate.** That's `/backlog-triage add` (or `/todo`). Use those if the user has already decided.
- **Skipping the hard-rule check.** Not every idea conflicts, but the ones that do are exactly the ones that look most attractive at first glance.
- **Generic pros/cons.** "It would improve UX" is not a pro. "It would unlock the 345 near-hardgate companies (ticket 008) by making the contribute flow shareable" is.
- **Demo gold-plating.** If a sketch is enough, write a sketch. Three boxes and a button beats a polished prototype that takes 15 minutes.
- **Suggesting a competitor's feature.** Beton Kemon's differentiator is the modal pay date + delay rate, gated. Random "Glassdoor has X" is not an argument; "this would help our specific cold-start problem" is.
- **Forgetting the user-feedback memory.** Do not propose moderation mechanisms that imply identifiers we don't store.

## Tone

Direct. Skeptical. Specific to this codebase. After debate: 8-15 lines max. The user wants a useful editor, not a TED talk.

## Gotchas

- **Don't write a ticket inside `/ideate`.** The value of this skill is the debate, not the artifact. Converting to a ticket happens via mode `commit` handing off to `/backlog-triage` - never edit `tasks/open/*.md` directly from this flow.
- **Surface hard rules early in the debate, don't bury them.** The rules from `AGENTS.md` (no PII on `salary_entries`, >=5 hardgate, no Gemini in user request path, no em dash, single canonical English data form) are non-negotiable. If an idea brushes one, name the rule in the first pass - don't let the user fall in love with a proposal that will be killed at commit time.
- **Demos must follow the production rules too.** No emoji, no em dash, no purple, no Inter, no `system-ui` even in throwaway HTML at `out/ideate/*.html`. If the demo violates the rules, the user internalizes the violation and it leaks into the real implementation.
- **Don't propose Gemini in the user request path.** Gemini is admin/offline only (classification, role mapping, spam audit). Any idea that calls Gemini from `/api/*` on a user request fails the architecture check immediately.
- **Regex-based ideas for spam / dedup under-flag by 5x or more.** If the proposal is "regex out bros, tora, etc.", push back: LLM-as-judge with `gemini-2.5-pro` is the established pattern. "More probably will be - bros, or tora" is the canonical example of regex blindspots.
- **Overlap check is not optional.** Duplicate tickets are a real failure mode here; always grep `tasks/open/*.md` and `tasks/done/*.md` before recommending File.

## Related

- `.claude/skills/backlog-triage/` - downstream when a ticket is the right outcome
- `.claude/skills/todo/` - even faster path for trivially-good adds (skip ideate)
- `tasks/README.md` - frontmatter conventions
- `AGENTS.md` - hard rules (cite by number when they trigger)
- `docs/adr/` - the why behind the rules
- `docs/product/` - user journey context for any flow-touching idea
- `docs/architecture.md` - cross-cutting context

## Ask at forks

If the work hits a real decision (architecture, scope, ticket-shape, user-facing copy/UX), use `AskUserQuestion` with concrete options + a recommendation - even in auto mode. See AGENTS.md "Working norms".

## Self-update on better patterns

If this session surfaces a better way of doing things (a new fork to flag, a tool that should have been preferred, a recurring confusion worth a paragraph), propose the edit to this skill / `AGENTS.md` / `docs/*.md` *in the same session*. Do not stash in per-agent memory. See AGENTS.md "Working norms".
