# SDLC: working a ticket end to end

The lifecycle of a single ticket from `tasks/open/` to merged + deployed. Designed for a one-person team plus AI agents - the flow is mandatory enough to keep quality consistent, light enough that a 30-minute task doesn't become a 4-hour ceremony.

For *why* each step exists, see the references inline. For commit + branching mechanics, see `CONTRIBUTING.md`. For testing specifics, see `docs/testing.md`. For deployment + rollback, see `docs/operations.md`.

## TL;DR

```
Pick ticket -> Read user journey -> Plan -> Implement
            -> Test -> Doc-sync -> Commit -> Close ticket -> Verify deploy
```

Steps are numbered below. Some are skippable for trivial work (typo fixes, copy tweaks); the heuristic is at each step.

---

## 1. Pick a ticket

- Run `/backlog-triage next` (or read `tasks/open/` directly). Preference: `priority: existential` first, `high` next, then by ticket number.
- Flip `status: open` -> `status: in-progress` in the ticket's frontmatter. Do not move the file yet (still in `tasks/open/`).
- Keep in-progress to 1-2 tickets max. If you're at 2, finish one before starting another.

**Skip-allowed for:** drive-by typo / copy fixes that don't have a ticket. Open a one-line ticket inline if useful, otherwise just commit.

## 2. Read the relevant user journey doc

The single most-skipped step, and the one that prevents the most regressions.

- If the ticket touches a user-facing flow, open `docs/product/<flow>.md` and read it. End-to-end. Even if you "wrote it last week".
- For admin work, `docs/product/admin-moderation.md`.
- For DB / pool work, `docs/database.md`.
- For Next.js framework questions, `docs/next-js-16.md`.
- Note any business logic, edge cases, or hard rules that apply.

**Why:** the journey docs encode invariants that aren't visible from the code alone (hardgate framing, copy rules, soft rate-limits, locale conventions). Discover them now, not in code review.

**Skip-allowed for:** isolated infra / dep bumps / scripts that don't touch a user-visible flow.

## 3. Check the design system if UI changes

If the ticket touches anything visible:

- Open `docs/design-system.md`. Confirm the design tokens you'll use exist in `src/app/globals.css` under `--bk-*`. Don't introduce new colors, radii, or motion durations without an explicit decision.
- Open the relevant mockup in `design/All Screens/` (or the print PDF) to anchor the visual.
- Check the spacing scale (4 / 8 / 12 / 16 / 24 / 32 / 48 / 64). No other values.

**Skip-allowed for:** non-UI changes.

## 4. Plan (lightweight)

- Sketch the change in your head or in 3-5 lines of plain text. Files to touch, schema changes, expected diff size.
- For non-trivial work, drop the plan into a `## Plan` section on the ticket itself - useful when it gets picked up later or someone else continues.
- Identify whether the change requires a new ADR (`docs/adr/`). Heuristic: if you're changing a hard rule in `AGENTS.md` or making a load-bearing trade-off, yes. Otherwise, no.

**Skip-allowed for:** trivial edits where the plan is self-evident.

## 5. Implement

- Edit code. Keep the diff scoped to what the ticket actually requires - no drive-by refactors, no "while I'm here" cleanups (per `AGENTS.md` "Doing tasks").
- If the change introduces a new server action or API route, design with the existing patterns (idempotency, transactional aggregate recompute, cache tag invalidation) - don't fork.
- If you're touching admin server actions, re-read `docs/admin.md` ADR-0005 before moving any `revalidatePath` calls.

## 6. Test

Per `docs/testing.md`:

- **Pure functions** in `src/lib/`: add or update a Vitest unit test. Run `npm test`.
- **User-flow change:** the Playwright suite is currently removed (rebuild tracked in `tasks/open/009-expand-e2e-coverage.md`). Manually verify the golden path with `npm run dev`; if you have appetite, contribute to the rebuild instead of writing a one-off spec.
- **Admin server-action change:** local DB-only scripts (`scripts/test-batch-*.mts`) are sanity checks; the **HTTP loadtest harness is the ship signal**. See `docs/admin.md`.
- **UI change on a route you can run:** `npm run dev` and verify the golden path in a browser. Don't claim "works" if you didn't open the page.

## 7. Doc-sync

Run the `/doc-sync` skill (or check manually against `CONTRIBUTING.md`'s file -> doc table). For any code change touching:

- A route (added / removed / behavior change) -> `docs/architecture.md` route table + the relevant `docs/product/<flow>.md`
- A schema (`supabase/migrations/`) -> `docs/architecture.md` schema section
- An admin flow -> `docs/admin.md` and `docs/product/admin-moderation.md`
- A hard rule in `AGENTS.md` -> add or update an ADR (`docs/adr/`)
- A pool / driver setting -> `docs/database.md`
- Design tokens / copy rules -> `docs/conventions.md`, possibly `docs/design-system.md`
- Test coverage shape -> `docs/testing.md`

If the change doesn't fit any row, no doc update needed. Don't fabricate doc edits.

## 8. Commit

Per the user's commit-per-todo memory and `CONTRIBUTING.md`:

- One logical change per commit. Imperative mood. Co-author trailer.
- Stage relevant files explicitly (don't `git add .`).
- Hooks run; don't `--no-verify`. If a hook fails, fix and recommit (do not amend - the hook failure means the commit didn't happen).

```bash
git add <files>
git commit -m "$(cat <<'EOF'
short imperative subject

Body explains *why*, not *what*. Cite ADRs or tickets if relevant.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

## 9. Close the ticket

**Iron rule: verify in dev before closing.** Typecheck + lint passing is NOT verification (see AGENTS.md "Never close a ticket without dev verification"). Before the `git mv`:
- Start `npm run dev` in the background.
- Navigate to each affected route via Playwright MCP (`mcp__plugin_playwright_playwright__browser_navigate` + `browser_snapshot`).
- Walk every acceptance bullet from the ticket and confirm visually.
- Scan `browser_console_messages` for runtime errors.
- For server-only changes (scripts, migrations, DB helpers with no UI surface), run the script/query and show the output instead.

Only then:
- `git mv tasks/open/NNN-slug.md tasks/done/NNN-slug.md`.
- Update frontmatter: `status: done`, add `closed: YYYY-MM-DD`.
- Optionally append a `## Resolution` section with commit SHA(s) and a one-line note.
- Commit the move alongside the work, or as a follow-up commit.

```bash
git mv tasks/open/NNN-slug.md tasks/done/NNN-slug.md
# edit frontmatter
git add tasks/done/NNN-slug.md
git commit -m "tasks: close NNN <title>"
```

## 10. Push and verify deploy

- `git push origin main`. Vercel auto-deploys.
- Open the Vercel deployments dashboard or `gh run watch` for the GitHub Actions CI.
- After deploy, verify with the post-deploy checks in `docs/operations.md`:
  - Cache hit on a known company page (`x-vercel-cache: HIT`).
  - Smoke search.
  - For admin changes, verify with the loadtest harness in `docs/admin.md`.
- If something burns, follow the incident protocol in `docs/operations.md`. Rollback in Vercel before fix-forward.

## When to short-circuit

- **Trivial change** (<10 lines, no behavioral impact): steps 1, 5, 6 (lint), 8, 10. Skip user-journey reading, plan, doc-sync.
- **Pure refactor with no behavioral change:** add an explicit "no behavior change" line to the commit body so reviewers know what to verify.
- **Hot-fix during incident:** stop the bleed first (rollback in Vercel), then resume the SDLC for the actual fix. Document the incident in `tasks/done/` even if there was no upfront ticket.

## When to expand

- **Cross-cutting change** (touches >5 flows or >10 files): add a `## Plan` to the ticket, possibly a new ADR. Consider a feature branch instead of pushing direct to main, and ask for a `/review` pass before merge.
- **Schema migration with risk:** read `docs/database.md` "column rename safety protocol" before touching anything. Migration + code deploy must land atomically.
- **New hard rule:** new ADR is mandatory. Update `AGENTS.md` to cite it.

## Anti-patterns

- **Skipping the user-journey read** because "I know this flow." This is the #1 source of regressions in repos with multi-month-old code.
- **Batching unrelated commits.** Breaks the audit log; makes rollback granular only at the file level.
- **Adding a new color or spacing value because it's "close enough".** Token discipline is what keeps the design coherent under AI editing.
- **Mocking the database in tests** (per user feedback memory). If a test needs a DB, hit a real Supabase or use the Playwright + postgres:17 service in CI.
- **Promising moderation mechanisms in copy that we don't actually implement** (per user feedback memory; especially anything implying we track identifiers).

## Related

- `CONTRIBUTING.md` - branching, commit style, doc-sync rule
- `tasks/README.md` - ticket frontmatter and movement rules
- `docs/product/` - user-journey docs (step 2)
- `docs/design-system.md` - tokens + mockups (step 3)
- `docs/testing.md` - what test to add when (step 6)
- `docs/operations.md` - deploy + rollback (step 10)
- `docs/adr/` - when to write a new ADR
