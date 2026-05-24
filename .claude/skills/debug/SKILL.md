---
name: debug
description: >
  Beton Kemon project-aware bug investigator for non-acute issues
  (a feature is broken, a number is wrong, a UI is off, an admin
  action does the wrong thing). Takes a symptom from the user,
  uses the CLAUDE.md "Read before changing code" routing table to
  auto-load the relevant docs/product/*.md and src/ slice,
  cross-references the suspected area against the project's hard
  rules and key invariants (pool=5, hardgate>=5, idempotency,
  cache tags, sync revalidatePath, statement_timeout), pulls
  runtime evidence from Sentry / Vercel / Supabase MCPs only when
  the symptom warrants it, then forms named hypotheses and
  verifies each one before any edit. Iron law: no fix until a
  named root cause is supported by evidence. Use when asked to
  "debug X", "why is Y broken", "this number looks wrong",
  "investigate this behavior", or "figure out what's happening
  with Z". For acute production fires (504 spike, site down) use
  /incident instead. For generic codebase-agnostic debugging use
  /investigate (gstack).
---

# /debug - project-aware bug investigation

Beton Kemon has enough surface area (public site, admin, API, aggregates, caching, i18n, moderation) that the failure mode for debugging is **starting in the wrong file**. This skill enforces a routing step before investigation: figure out which slice of the codebase + which docs apply, load them, then form hypotheses informed by the project's actual invariants.

It is the non-acute cousin of `/incident`. If the site is on fire right now, stop and use `/incident`. If a feature is misbehaving, a number looks off, an admin action did something unexpected, or a user-reported bug needs root-cause analysis, this is the skill.

**Iron law:** no fix until a named root cause is supported by evidence. A hypothesis is not a root cause. "It might be the cache" is not a root cause; "the `LANDING_CACHE_TAG` is not invalidated by `approveEntry` because the `updateTag` call is missing" is.

## Phase 0 - Intake (don't skip; bad intake = wrong slice)

If the user gave you a one-liner ("logos are broken", "the count is wrong"), ask the minimum to pick a slice. Aim for one short message with the questions bundled, not a back-and-forth.

Required:

- **Symptom in one sentence.** What did you see vs what did you expect?
- **Where.** URL / route / admin page / API endpoint / script. If multiple, pick the most reproducible.
- **When.** First noticed when? After a deploy? After a specific admin batch? "Always" is a valid answer.
- **Repro.** Steps if known. "Click X, see Y." If not known, say so explicitly so we know we're hunting blind.
- **Error text.** Any visible error, console log, Sentry link, or screenshot. Quote it verbatim.
- **Scope.** One row / one company / all companies / one locale / mobile only?

Skip questions you can answer from context. If the user already said "the company page for /c/aarong shows 0 entries but admin says 12 are approved", you have all six.

If the symptom is acute production (5xx, p95 timeout spike, "site is down") - **stop and tell the user to use `/incident`**. Don't start a non-acute investigation on a fire.

## Phase 1 - Route to the right slice

Use the `CLAUDE.md` "Read before changing code" routing table. The mapping is the contract; don't improvise.

| Symptom area                                        | Load these                                                                                                                                  |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Landing page, browse list, filters, sort, pagination | `docs/product/home.md` + `src/app/[locale]/page.tsx` + `src/components/{CompanyCard,FilterAccordion,SortControl,Pagination}.tsx` + `src/lib/companies.ts` |
| Search bar, autocomplete, role suggest              | `docs/product/search.md` + `src/app/api/search/` + `src/app/api/roles/suggest/` + `src/components/SearchBar.tsx` + `src/lib/common-roles.ts` |
| Company page (`/c/[slug]`), hero, role cards        | `docs/product/company-page.md` + `src/app/[locale]/c/[slug]/` + `src/lib/companies.ts` + `src/lib/aggregate.ts`                              |
| Contribute flow, `/api/submit`, auto-approve        | `docs/product/contribute.md` + `src/app/[locale]/contribute/` + `src/app/api/submit/` + `src/lib/aggregate.ts` + `src/lib/turnstile.ts`      |
| Onboarding, locale gate, trust copy                 | `docs/product/onboarding.md` + `src/app/[locale]/onboarding/` + `src/lib/onboarding.ts`                                                     |
| Admin moderation, queue, reports, mappings          | `docs/product/admin-moderation.md` + `docs/admin.md` + `src/app/admin/` + `src/lib/admin-guard.ts` + `src/lib/reports.ts`                    |
| Routing, caching, `cookies()` / `headers()`         | `docs/next-js-16.md` + the route file(s) + `src/i18n/`                                                                                      |
| Query latency, 504s, pool                           | `docs/database.md` + `src/lib/db.ts` (then escalate to `/incident` if acute)                                                                |
| Schema, migrations, aggregates                      | `docs/architecture.md` + `supabase/migrations/` + `src/lib/aggregate.ts`                                                                    |
| Design tokens, layout, mockups                      | `docs/design-system.md` + `docs/conventions.md`                                                                                             |
| Copy, i18n, locale-specific bugs                    | `docs/conventions.md` + `messages/{bn,en}.json`                                                                                             |
| Library / framework misbehavior (Next.js cache semantics, Tailwind v4 token resolution, `postgres.js` connection options, `next-intl` middleware, Resend / Turnstile SDK quirks) | **Context7 MCP** (`mcp__context7__resolve-library-id` -> `mcp__context7__query-docs`) before forming hypotheses from training data |
| Issue reported via GitHub issue or PR comment       | **GitHub MCP** (`mcp__github__issue_read`, `pull_request_read`, `search_code`) to load the original report verbatim                          |

If the symptom doesn't fit cleanly, pick the **two** most likely slices, load both, and announce which one you're starting with and why.

State the routing decision out loud:

> Slice: company-page. Loading docs/product/company-page.md, src/app/[locale]/c/[slug]/, src/lib/companies.ts, src/lib/aggregate.ts.

## Phase 2 - Cross-reference invariants

Before forming any hypothesis, walk the symptom against the project's hard rules and key invariants from `CLAUDE.md`. Many "bugs" are an invariant doing its job.

For the loaded slice, check the relevant invariants:

- **Aggregates / counts off** -> `>=5 hardgate` (ADR-0004). Below threshold deliberately shows locked-preview empty state. Is the count "wrong" because the company has fewer than 5 approved entries?
- **Submit double-counts or under-counts** -> `/api/submit` idempotency + transactional aggregate recompute (`src/lib/aggregate.ts`, `sql.begin(tx)`). Is the recompute partial? Is the idempotency key derivation correct?
- **Admin action's effect doesn't appear** -> `revalidatePath("/admin/queue")` is sync; cross-page invalidations via `after()` (ADR-0005). Did the action call `updateTag(LANDING_CACHE_TAG)`? Is the change wrapped in `unstable_cache` and waiting on tag invalidation?
- **504 / timeout / "feels slow"** -> pool max=5 (ADR-0006), DB statement_timeout=10s. If acute, switch to `/incident`. If consistent on one query, look at `pg_stat_statements`.
- **Admin batch silently serializes** -> client `CHUNK` in `src/app/admin/queue/Queue.tsx` must match `max=5` in `src/lib/db.ts`. Mismatch is the canonical cause.
- **PII shows up where it shouldn't** -> `salary_entries` schema bans IP/UA/fingerprint (ADR-0003). Schema-enforced; if PII is appearing, the source is elsewhere.
- **UI has em dash, emoji, purple, Inter, `system-ui`, Bangla numerals, Bangla company names** -> hard-rule violation (ADR-0001, ADR-0007), not a bug per se, but flag it.
- **Bangla / English mismatch** -> single canonical form (ADR-0007); UI chrome localizes, data doesn't.
- **`format.ts` import error** -> `src/lib/format.ts` is client-safe; if you imported a server-only module into it, that's the bug.
- **Production HTML probe returns 429** -> bot protection (use Playwright MCP), not a real bug.

If the invariant explains the behavior, **the symptom is correct behavior**. Tell the user, cite the ADR, and stop. Don't fix what isn't broken.

## Phase 3 - Form hypotheses with evidence pointers

For each plausible cause, write:

- **Hypothesis** (one sentence, names a specific file/function/query/config)
- **Verification** (the exact action that would prove or disprove it: a query, a log fetch, a file read, a repro step)

Bias the list toward cheapest verification first. Reading a file is cheaper than fetching logs is cheaper than running a query is cheaper than reproducing in the browser.

Example for "company page shows 0 entries but admin shows 12 approved":

```
H1: aggregate row in `companies` table is stale (recompute didn't run)
   Verify: select * from companies where slug = 'aarong'; compare
           approved_entry_count to count(*) from salary_entries
           where company_id = ... and status = 'approved'.

H2: cache is stale (LANDING_CACHE_TAG not invalidated after recent approvals)
   Verify: read src/app/admin/actions.ts approveEntry path; check
           updateTag(LANDING_CACHE_TAG) is called.

H3: hardgate suppression - company actually has <5 approved
   Verify: same SQL as H1; if approved < 5, locked-preview is correct.
```

Order matters. H3 is cheapest (one query), so run it first.

## Phase 4 - Verify (this is where most time goes)

Run verifications one at a time. After each, state the verdict explicitly:

> H3 verdict: 12 approved entries confirmed. Hardgate is satisfied. H3 disproved.

Use the right tool:

- **File reads** for "is this code path doing X" -> Read tool
- **SQL** for "what does the data actually look like" -> Supabase MCP `execute_sql` (project_id `hasfagvyjbchoprrwgaj`)
- **Runtime logs** for "did this happen in production and what was the stack" -> `/website-logs-10m` or `/website-logs-2h`, or Sentry MCP directly (`search_issues`, `search_events`)
- **Vercel build / deploy state** -> Vercel MCP (`get_deployment`, `get_deployment_build_logs`)
- **Live page state** -> Playwright MCP (`browser_navigate`, `browser_snapshot`, `browser_network_requests`). Production HTML 429s curl - always Playwright for prod URLs.
- **API JSON** -> curl is fine for `/api/*` (sometimes), Playwright fallback on 429.

Stop the moment a hypothesis is confirmed with a named cause that fully explains the symptom. Don't keep verifying for completeness.

## Phase 5 - Indictment

Write a short summary in the same shape as `/incident`:

```
Indictment: <named root cause>

Evidence:
  - <verification 1 result>
  - <verification 2 result>
  - <code reference: file:line>

Affects: <which routes / data / users>
Severity: <cosmetic | wrong-data | broken-flow | revenue/trust>
```

If you can't reach an indictment - the verifications are inconclusive, the symptom doesn't repro, or evidence points two ways - **say so** and propose the next experiment instead of guessing. Marking unknown unknowns is more valuable than a confident wrong fix.

## Phase 6 - Fix (only after indictment)

Now propose the fix. Keep it minimal:

- Touch only the file(s) the indictment names. Don't bundle "while I'm here" cleanup.
- Re-read the relevant `docs/product/*.md` and any cited ADR before editing - the fix has to stay inside the rule.
- If the fix touches **routes, schema, hard rules, or pool config**, run `/doc-sync` before commit (per `CLAUDE.md` rule 12).
- If the fix changes a hard rule or invariant, write a new ADR in `docs/adr/` in the same commit.
- Add a regression test in `e2e/` or `src/**/__tests__/` if the bug was reachable from a user flow. See `docs/testing.md`.

Per the user's standing instruction: one commit per completed todo, with the Co-Authored-By footer. If the fix is multi-step (write test, fix code, update doc), each step is its own commit.

If the fix is non-obvious or touches a hot path (cache, pool, aggregate, submit), get a `/codex` second opinion before landing.

## Phase 7 - Close out

- File a `tasks/done/debug-<YYYY-MM-DD>-<slug>.md` only if the bug was non-trivial (root cause was subtle, took >30 min, or the fix changed an invariant). Trivial bugs don't need a postmortem; the commit message is the record.
- If you found related issues during investigation that aren't part of this fix, file them as new tickets in `tasks/open/` via `/backlog-triage` rather than expanding scope.
- If the bug pattern is likely to recur (e.g. "we keep forgetting to call `updateTag`"), update the relevant `docs/*.md` or the affected skill so future agents catch it.

## Anti-patterns

- **Skipping intake.** "Logos are broken" without scope (one company? all? mobile only? after which deploy?) sends you into the wrong slice. 30 seconds of intake saves an hour of wandering.
- **Loading "everything" instead of routing.** The point of the routing table is to keep the slice small. If you load all of `src/`, you're just doing keyword search.
- **Fixing without indicting.** The temptation is to spot a "smell" and patch it. If the smell didn't cause this bug, you're shipping noise and may mask the real cause.
- **Treating Sentry / Vercel logs as cause.** Logs are evidence of effect, not cause. A `Vercel Runtime Timeout` doesn't tell you why; it tells you the lambda was killed.
- **Ignoring the invariants.** Half of "bugs" filed against this project are the hardgate, idempotency, or cache tag doing exactly what they should. Walk the invariants before forming a hypothesis.
- **Bundling cleanup.** `CLAUDE.md` is explicit: don't add features, refactors, or "while I'm here" changes during a bugfix. One bug, one fix, one commit.
- **Curl against production HTML.** It returns 429. Use Playwright. (`/api/*` JSON endpoints sometimes pass; fall back to Playwright on 429.)
- **Assuming memory is current.** `pg_stat_statements`, `show max_connections`, the live `salary_entries` count - all change. Verify, don't recall.

## Tone

Curious and methodical. The user is not under fire (that's `/incident`'s job); they want a careful read of what's actually happening. Each phase has a clear deliverable - don't move on until it's named.

State what you're doing as you go: "Routing to company-page slice", "Walking invariants", "H3 verdict: disproved", "Indictment ready". The user should be able to follow the investigation without scrolling through tool calls.

## Gotchas

- **Don't reach for `/incident` for non-acute bugs.** `/debug` is the project-aware one; `/incident` is the on-fire walker. If the user said "this number looks wrong" or "the count is off", you are in the right skill - resist the urge to escalate.
- **Check the `CLAUDE.md` routing table FIRST before reading code.** The table points you to the relevant `docs/product/*.md`. Loading 10 random `src/` files and pattern-matching is the canonical wrong start - it burns tokens and lands you in the wrong slice.
- **Form hypotheses BEFORE wide file reads.** If you have no named hypothesis, you have nothing to verify - you're just touring the codebase. Phase 3 is in front of Phase 4 for a reason.
- **No fix without a named root cause supported by evidence.** "It might be the cache" is not a root cause. The user has explicitly asked agents to "be truthful and show me the reality" - reporting "fixed" without a verification step that exercises the bug is a violation, not a shortcut.
- **Production HTML probes need Playwright; close the window when done.** `curl` against `betonkemon.com/*` returns 429. After any Playwright session for verification, `browser_close` before reporting done - leaving the Chromium window open is a recurring complaint.

## Related

- `.claude/skills/incident` - the acute-production cousin; switch to it if the site is on fire
- `.claude/skills/website-logs-10m`, `website-logs-2h` - multi-source log fans used in Phase 4
- `.claude/skills/doc-sync` - run before committing if the fix touches routes/schema/hard rules/pool
- `.claude/skills/backlog-triage` - file related issues found during investigation here
- `/investigate` (gstack) - generic, codebase-agnostic debugger; use only if `/debug` doesn't apply
- `/codex` (gstack) - second opinion on non-obvious fixes
- `CLAUDE.md` - the routing table and invariants this skill enforces
- `docs/adr/` - the "why" behind every invariant

## Ask at forks

If the work hits a real decision (architecture, scope, ticket-shape, user-facing copy/UX), use `AskUserQuestion` with concrete options + a recommendation - even in auto mode. See AGENTS.md "Working norms".

## Self-update on better patterns

If this session surfaces a better way of doing things (a new fork to flag, a tool that should have been preferred, a recurring confusion worth a paragraph), propose the edit to this skill / `AGENTS.md` / `docs/*.md` *in the same session*. Do not stash in per-agent memory. See AGENTS.md "Working norms".
