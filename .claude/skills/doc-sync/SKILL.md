---
name: doc-sync
description: >
  Beton Kemon doc-sync checker. Maps changed file paths to the docs/*.md
  that should be updated alongside, per the doc-sync rule in
  CONTRIBUTING.md. Use proactively after any non-trivial code change,
  or when asked to "check docs", "doc sync", "what docs need updating",
  "before commit", or "did I miss any docs".
---

# /doc-sync - Documentation Sync Checker

Beton Kemon enforces a **doc-sync rule**: any code change touching routes, schema, hard rules, or pool config must update the relevant `docs/*.md` in the same commit. This skill catches misses before commit.

## How it works

Inspect `git diff --name-only HEAD` (or staged: `git diff --cached --name-only`) and report which docs to check.

## Mapping table

| Changed file pattern | Doc to update |
| --- | --- |
| `src/app/**/page.tsx` (new/removed routes) | `docs/architecture.md` route table |
| `src/app/**/route.ts` (new/removed APIs) | `docs/architecture.md` API table |
| `src/app/admin/actions.ts` | `docs/admin.md` (esp. revalidatePath rules; ADR-0005) |
| `src/app/admin/**` | `docs/admin.md` |
| `src/lib/db.ts` | `docs/database.md` + likely a new ADR (`docs/adr/`) |
| `src/lib/aggregate.ts` | `docs/architecture.md` (aggregate recompute section) |
| `src/lib/companies.ts` | `docs/architecture.md` (caching section) |
| `supabase/migrations/*.sql` | `docs/architecture.md` schema table; possibly `docs/database.md` |
| `messages/*.json` | only if a new feature added; check for em-dash violations |
| `src/components/Icon.tsx` | `docs/conventions.md` icon section if set changes |
| `src/app/globals.css` (`--bk-*` tokens) | `docs/conventions.md` tokens |
| `next.config.ts`, `vercel.json` | `docs/operations.md` |
| Any change to `AGENTS.md` hard rules | new ADR in `docs/adr/` |
| `package.json` deps | flag for review; some affect security/license posture |

## Workflow

1. Run `git diff --name-only HEAD` (or `--cached`).
2. Cross-reference against the mapping table.
3. For each match: check if the corresponding doc was also modified.
4. Report:
   - **MISSING** docs that should be touched but aren't.
   - **OK** docs that are appropriately updated.
   - **MAYBE** ambiguous cases (e.g., a small `actions.ts` tweak that doesn't change behavior - judgment call).

## Output

```
doc-sync report
---
MISSING:
  src/app/admin/actions.ts changed -> docs/admin.md NOT touched
  Suggest: review the revalidatePath section; ADR-0005 still holds?

OK:
  src/lib/companies.ts -> docs/architecture.md updated

MAYBE:
  src/components/Icon.tsx changed - did the icon set grow/shrink?
  If yes: docs/conventions.md icon list. If no: ignore.
```

## When to skip

- Pure typo fixes
- Comment-only changes
- Lockfile-only changes (`package-lock.json`)
- Test-only changes (`*.test.ts`, `e2e/*.spec.ts`)

These don't trigger the rule. Don't over-report.

## Tone

Terse. The user is about to commit; deliver the verdict in <10 lines unless there are real misses.

## Gotchas

- **`CONTRIBUTING.md` is the source of truth, not memory.** If the mapping table here drifts from `CONTRIBUTING.md`, the latter wins. Re-read it when in doubt rather than trusting recall.
- **Don't invent target docs.** If a changed file pattern is not in the table, flag it as MAYBE for human review - never suggest "update docs/foo.md" without confirming the doc exists. Hallucinated doc paths waste the user's pre-commit minute.
- **Migrations land in a specific order vs code.** If the diff contains both `supabase/migrations/*.sql` and code that depends on the new schema, call out the deploy ordering explicitly - applying the migration before the code deploys is a known way to "leave production broken."
- **Lockfile + test-only diffs are not free passes for everything else.** Skip the doc check only when the diff is *exclusively* in the skip list. A diff that mixes `*.test.ts` with `src/lib/db.ts` still triggers the db.ts rule.
- **ADR additions require their own doc.** If a hard rule in `AGENTS.md` shifted, the matching `docs/adr/NNNN-*.md` must land in the same commit. "We'll write the ADR later" is the canonical way it never happens.

## Related

- `CONTRIBUTING.md` (the rule itself)
- `AGENTS.md` hard rule #12

## Ask at forks

If the work hits a real decision (architecture, scope, ticket-shape, user-facing copy/UX), use `AskUserQuestion` with concrete options + a recommendation - even in auto mode. See AGENTS.md "Working norms".

## Self-update on better patterns

If this session surfaces a better way of doing things (a new fork to flag, a tool that should have been preferred, a recurring confusion worth a paragraph), propose the edit to this skill / `AGENTS.md` / `docs/*.md` *in the same session*. Do not stash in per-agent memory. See AGENTS.md "Working norms".
