---
name: website-logs-2h
description: >
  Beton Kemon multi-source production audit over a 2-hour window. Fans out
  in parallel to Vercel runtime logs, Sentry issues/events with release
  tagging, and Supabase Postgres + pooler logs + pg_stat_statements.
  Optimized for trend detection, billing-impact analysis, post-deploy
  verification, and disambiguating truncated error classes by cross-
  referencing the same window across all three surfaces. Trigger with
  /website-logs-2h. For "is the site on fire RIGHT NOW", use
  /website-logs-10m. (Replaces the older /vercel-logs-2h, which only saw
  Vercel.)
---

# /website-logs-2h - Beton Kemon Production Audit (2h trend deep dive, all sources)

You are auditing **2 hours of production logs across three surfaces** for **Beton Kemon** (`betonkemon.com`):

1. **Vercel runtime logs** - request edge view (status codes, function timeouts, edge cache, billing categories)
2. **Sentry** - app-error view (full untruncated stack traces, issue grouping, release tagging by `VERCEL_GIT_COMMIT_SHA`, frequency-over-time)
3. **Supabase logs + `pg_stat_statements`** - data-layer view (slow queries, pool exhaustion, statement timeouts, egress)

The goal is to understand **trends**, **billing impact**, and **post-deploy health** - not to triage a live fire (use `/website-logs-10m` for that).

**Hardcoded coordinates (never prompt):**
- Vercel `projectId`: `prj_4jL6tTaDtN3uo339kSu3XCvcS8vE`
- Vercel `teamId`: `team_Y4Oz01XhgpTFPDVclHoY5GwL`
- Sentry `organizationSlug`: `betonkemon`
- Sentry `projectSlug`: `javascript-nextjs`
- Supabase `project_id`: `hasfagvyjbchoprrwgaj`

---

## What this skill is for

| Use this when | Use /website-logs-10m when |
|---|---|
| "How did we do today / this hour / since the last deploy?" | "Something is broken right now" |
| "Did our perf fix actually work?" | "I just deployed, did it land cleanly?" |
| "What's burning my Vercel / Supabase budget?" | "User reported error 30 sec ago" |
| "Show me the trend across the last cycle" | "Is the site on fire?" |
| Post-deploy verification (give it 30+ min to settle) | Real-time triage |
| "Which Sentry issues regressed after release [SHA]?" | One-shot fire check |

---

## Cardinal rules - read before starting

0. **ALL TIMESTAMPS IN THE REPORT MUST BE BDT (UTC+6).** Vercel, Supabase, and Sentry MCPs return UTC. Convert every single time you cite (table rows, burst windows, deploy times, "what's working ✓" rows, "since the last deploy" math). Never paste a raw UTC time into the report without converting. Format: `22:21 BDT` (or `2026-05-10 22:21 BDT` if the date matters). If the source matters for an operator who'll cross-check the dashboard, show both: `16:21 UTC (22:21 BDT)`. The user is in Bangladesh and reads dashboards in BDT — raw UTC forces them to do +6 in their head on every line. This is non-negotiable; reports that mix UTC and BDT get sent back.

1. **Vercel `runtime_logs` truncates the message column.** Sentry doesn't. A Vercel line ending `[Error: An error occurred i...` is ambiguous; Sentry has the full digest and stack. Always cross-reference.

2. **Sampling bias is real.** Vercel `limit: 100` over `since: "2h"` returns only the most recent 100 matching rows, biased toward the tail. With 50+ requests/sec, that's <0.5% of traffic. NEVER report absolute counts as if they're total volume. Always report **rates** and **proportions**. Sentry's `events()` aggregation is more truthful for frequency; prefer it for trend claims.

3. **One MCP call gives one slice of one filter.** Each filter combination is its own data point.

4. **Distinguish error classes by Sentry digest, not Vercel truncated message.** Production Next.js errors all read identically when truncated. The actual digest (`DYNAMIC_SERVER_USAGE`, `SERIALIZATION_ERROR`, etc.) is in the Sentry event, not the Vercel log.

5. **Steady-state vs spike: always check.** Run the same filter at multiple sub-windows (10m / 30m / 1h / 2h) and compare densities. Sentry stats over the same windows confirm.

6. **Cross-reference with the user's dashboard view.** A wall-of-red screenshot beats any "looks rare" conclusion from sampled logs.

7. **Tie findings to deployments AND releases.** Vercel: `Deployment: dpl_xxx`. Sentry: `release: <git sha>`. A 2h window may span a deploy; correlate errors against the boundary.

8. **Supabase egress / connection ceilings are billing+reliability mixed.** A 2h spike in egress can both blow the bill and produce pool-exhaustion symptoms. Check Supabase usage when Vercel 504s show up.

---

## Phase 1 - Multi-source parallel fetch

Send all of the following as parallel tool calls in one message. **Track every `Showing N+ logs` indicator** - if N hits the limit, that filter is volume-saturated and the actual count is unbounded.

### Group A: Vercel - density baseline (4 windows, same filter)
1. `level: ["error", "fatal"]`, `since: "10m"`, `limit: 100`
2. same - `since: "30m"`
3. same - `since: "1h"`
4. same - `since: "2h"`

Steady-vs-spike: if errors-10m hits 100 → ≥10 errors/min sustained → 🔴 firehose. If errors-1h hits 100 but 10m is much lower → trailing-off. If 2h hits 100 but 1h doesn't → fixed in last hour.

### Group B: Vercel - source breakdown (30m)
5. `source: ["serverless"]`, `since: "30m"`, `limit: 100`
6. `source: ["edge-middleware"]`, `since: "30m"`, `limit: 100`
7. `source: ["static"]`, `since: "30m"`, `limit: 100` - cache HITs

### Group C: Vercel - status code buckets (1h)
8. `statusCode: "500"`, `since: "1h"`, `limit: 100`
9. `statusCode: "504"`, `since: "1h"`, `limit: 100` - the pool-exhaustion fingerprint
10. `statusCode: "404"`, `since: "2h"`, `limit: 100` - bot probes / dead routes
11. `statusCode: "429"`, `since: "1h"`, `limit: 100` - rate-limit hits

### Group D: Vercel - known signatures (full message via query)
12. `query: "Vercel Runtime Timeout"`, `since: "2h"`, `limit: 50`
13. `query: "ECONNREFUSED"`, `since: "2h"`, `limit: 30`
14. `query: "/api/submit"`, `source: ["serverless"]`, `since: "2h"`, `limit: 100`
15. `query: "/api/search"`, `source: ["serverless"]`, `since: "1h"`, `limit: 50`
16. `query: "/[locale]/c/"`, `since: "30m"`, `limit: 100`
17. `query: "revalidating cache with key"`, `since: "1h"`, `limit: 30` - ISR noise check

### Group E: Sentry - issue trends and release correlation
18. `mcp__sentry__search_issues` - `naturalLanguageQuery: "issues seen in the last 2 hours sorted by event count desc"`, `limit: 25`
19. `mcp__sentry__search_issues` - `naturalLanguageQuery: "new issues first seen in the last 2 hours"`, `limit: 25` - regression signals
20. `mcp__sentry__search_events` - `naturalLanguageQuery: "errors in the last 2 hours grouped by release"`, `limit: 50` - did a specific release introduce errors?
21. `mcp__sentry__search_events` - `naturalLanguageQuery: "errors in the last 2 hours grouped by transaction (path)"`, `limit: 50`

### Group F: Supabase - data layer over 2h
22. `mcp__supabase__get_logs` - `service: "postgres"` - slow queries, statement_timeout cancellations
23. `mcp__supabase__execute_sql` - `select date_trunc('minute', terminated_at) bucket, count(*) from public.wedged_backend_log where terminated_at > now() - interval '2 hours' group by 1 order by 1 desc;` - Supavisor backend kills by minute (proxy for slot starvation rate). **NOTE:** Supabase MCP `get_logs` does not expose pooler logs (`service` only accepts `api | branch-action | postgres | edge-function | auth | storage | realtime`). For `EMAXCONN` strings, search Vercel logs (`query: "EMAXCONN"`) - the postgres.js client throws it server-side, so it surfaces in serverless function logs.
24. `mcp__supabase__get_logs` - `service: "api"` - PostgREST errors
25. `mcp__supabase__execute_sql` - `select query, calls, mean_exec_time, max_exec_time, total_exec_time from pg_stat_statements where calls > 100 order by total_exec_time desc limit 15;` - billing-relevant: total time = compute spend
26. `mcp__supabase__execute_sql` - `select count(*), state, application_name from pg_stat_activity group by 2,3 order by 1 desc;` - current slot usage
27. `mcp__supabase__get_advisors` - `type: "performance"` - any new advisor warnings

Send Groups A-F all in parallel. Do not wait between them.

---

## Phase 2 - Disambiguate every error class (Sentry-first)

For each unique error pattern detected in Vercel:

1. **Look it up in Sentry.** The Sentry issue has the full message, digest, frame stack, breadcrumbs, and release tag. If Sentry doesn't have it, the error is platform-level (504 timeout, function killed before throw) - go to Phase 2b.

2. **Pull a representative event.** `mcp__sentry__search_issue_events` for the issue ID; or `mcp__sentry__analyze_issue_with_seer` for AI root-cause analysis on non-obvious errors.

3. **Note the release tag.** `release: <VERCEL_GIT_COMMIT_SHA>`. If the issue's first-seen release matches a recent deploy and the issue didn't exist before, that's a regression.

4. **Never group two truncated Vercel messages as the same error without proof.** Use Sentry's grouping; that's what it's for.

5. **Calculate per-path rate, not aggregate count.** "60 errors in 2h on /c/[slug]" is meaningful. "60 errors in 2h" alone is not.

### Phase 2b: Platform-level errors (Vercel sees, Sentry doesn't)

If Vercel has 504 clusters but Sentry is silent for the same window, the lambda was killed before it could throw. Run the **Pool exhaustion diagnostic protocol** below.

---

## Phase 3 - Build the trend table

| Error class (Sentry digest) | Vercel 10m | 30m | 1h | 2h | Sentry events 2h | First seen release | Trend |
|---|---|---|---|---|---|---|---|
| `DYNAMIC_SERVER_USAGE` | ... | ... | ... | ... | ... | sha xxx | rising / steady / falling / stopped |
| `Vercel Runtime Timeout` (no Sentry, platform-level) | ... | ... | ... | ... | n/a | n/a | ... |

If a row reads `0 / 0 / 0 / 60`, that class **stopped** firing - confirmed-fixed. If `15 / 40 / 75 / 100+`, the issue is current and growing.

For paths:

| Path | Vercel sample requests | Vercel sample errors | Error rate | Sentry events | Severity |
|---|---|---|---|---|---|
| `/[locale]/c/[slug]` | ... | ... | ...% | ... | 🔴/🟡/🟢 |
| `/api/search` | ... | ... | ...% | ... | ... |
| `/[locale]/contribute` | ... | ... | ...% | ... | ... |

>5% error rate = user-impacting. >25% = page is functionally broken.

---

## Phase 4 - Billing impact assessment

Map findings across **both** Vercel and Supabase billing. Compute sampled rates, not absolute counts.

### Vercel billing

| Line | Driven by | Sampled rate | Trend vs prior | Severity |
|---|---|---|---|---|
| Function Invocations | Every serverless hit | (Group B count / 30) × 60/h | ... | 🔴/🟡/🟢 |
| Fluid Active CPU | Slow durations, esp. 504s | (504 count / 1h) × 10s = burned s/h | ... | ... |
| Observability Events | Every log line incl. framework noise | Sum × 30 (typical 30 lines/error) | ... | ... |
| Edge Requests | Middleware-hit requests | (middleware count / 30) × 60 | ... | ... |
| ISR Writes | revalidatePath/Tag firing | "revalidating cache with key" count | ... | ... |

### Supabase billing

| Line | Driven by | Sampled rate | Trend vs prior | Severity |
|---|---|---|---|---|
| DB compute hours | mean × calls in `pg_stat_statements` | sum(total_exec_time)/h on top queries | ... | 🔴/🟡/🟢 |
| Egress | Row volume × hits | dashboard `get_project` egress field, or estimate from query rows | ... | ... |
| Disk size | Migrations + entry growth | `mcp__supabase__list_tables` row count delta | ... | ... |
| Edge function invocations | If we ever ship one | n/a currently | ... | ... |

Severity (BK on Pro+Micro, $20 included credit):
- 🔴 **Still leaking**: rate is on track for >$10/cycle on this line
- 🟡 **Watch**: $3-10/cycle
- 🟢 **Under control**: <$3/cycle

---

## Phase 5 - Cross-reference and reality-check

1. **Identify the latest deployment** (Vercel) **and release** (Sentry) from the log output. Confirm they match.

2. **If user shared a dashboard screenshot**, that's ground truth.

3. **Compare against `PERF_AUDIT_PROGRESS.md` if it exists.** Did we say we fixed something? The trend table makes it easy: rows where 2h has count but 10m has 0.

4. **Sanity check rates:**
   - ~17K visitors/day as of 2026-05-07 (see `project_launch_traffic` memory)
   - Expected total RPS: 30-60 sustained, peaking BD evening
   - Expected error rate baseline: <1% for healthy app

5. **If sampling gives unclear picture, say so.** "MCP sampling shows X but volume in dashboard suggests Y" beats committing to one.

---

## Phase 6 - Report format

```markdown
## Website Audit (2h) - [BDT timestamp]
**Window samples:** 10m / 30m / 1h / 2h
**Latest Vercel deployment seen:** [dpl_xxx]
**Latest Sentry release seen:** [git sha]
**Deployment + release matched / lagged?** [yes / no / unknown]

### Sampling notes
- Vercel errors-10m returned [X] entries → [steady firehose ≥10/min | spiky | low-rate | clean]
- Sentry issues seen 2h: [N], new: [M]
- Supabase pooler: [N] EMAXCONN events / 2h, top app_name slot holders: [...]
- [Any volume-saturated filter is a lower bound]

### Error class trend table
| Class (Sentry digest) | Vercel 10m | 30m | 1h | 2h | Sentry events | Release | Trend | Severity |
|---|---|---|---|---|---|---|---|---|

### Path-level rate table
| Path | Vercel reqs | Vercel errs | Rate | Sentry events | Severity |
|---|---|---|---|---|---|

### Vercel billing signals
| Line | Sampled rate | Trend | Severity |
|---|---|---|---|

### Supabase billing signals
| Line | Sampled rate | Trend | Severity |
|---|---|---|---|

### Per error class (Sentry full message captured, not truncated)
**[ERROR_NAME / digest]**
- Full message: [from Sentry]
- Affects paths: [Vercel + Sentry intersection]
- Rate: ~[N]/min sampled
- First-seen release: [SHA]
- Likely cause: [one line]
- Recommended fix: [file:line if known]

### What's working ✓
[Items confirmed with evidence: classes that dropped to 0 in 10m vs 2h]

### Still broken / new issues ⚠
- **[path]**: [error class], [rate], [user impact]

### Top 3 actions ranked by user impact + billing impact
1. ...
2. ...
3. ...

### Open questions for the operator
- [Anything no MCP can answer]
```

---

## What NOT to do

- ❌ Do not collapse two distinct truncated Vercel messages into one error class. Use Sentry's grouping.
- ❌ Do not report "60 errors / 2h" as if it's the total. State sampling.
- ❌ Do not say "occasional spikes" when sampled 100 rows from 2h. Dashboard sees firehose; MCP sees slice.
- ❌ Do not skip the Sentry digest check.
- ❌ Do not mix EMAXCONN (DB pool) with `DYNAMIC_SERVER_USAGE` (framework) - different layers, different fixes.
- ❌ Do not present billing impact in absolute dollars without the rate. Cost = rate × time × unit price.
- ❌ Do not treat `[error] revalidating cache with key...` as an actual error.

---

## Beton Kemon-specific signatures

| Pattern | Where it shows | Meaning | Action |
|---|---|---|---|
| `Vercel Runtime Timeout Error` cluster across multiple paths AND Sentry silent | Vercel only | Pool exhaustion. Lambda killed at 10s waiting for Supavisor slot - app never threw, so Sentry blind. | Pool exhaustion protocol below. **Don't EXPLAIN ANALYZE first**; check `pg_stat_statements` and `show max_connections`. |
| `(EMAXCONN) max client connections reached` | Vercel + Supabase pooler | postgres.js client-side error. Function survived past 20s `connect_timeout`. Rare; usually masked by 10s Vercel cap. | Lower `max` per pool, raise compute, or cache hot endpoint. |
| `DYNAMIC_SERVER_USAGE` digest | Sentry | Layout/page used `cookies()`, `headers()`, or `searchParams` in ISR-cached route. | Find dynamic API call in layout/page tree, refactor or `force-dynamic`. |
| Single `canceling statement due to statement timeout` | Supabase Postgres | Background noise (slow ANALYZE). | Ignore unless clustered. |
| `revalidating cache with key` at error level | Vercel | Normal ISR background work. | Ignore unless flooding Observability cost. |
| 404 on `/api/companies`, `/meta.json` | Vercel | Bot probes for non-existent endpoints. | Block at edge or in `next.config.ts` rewrites. |
| 304 on `/icon`, `/apple-icon`, `/[locale]/about`, `/legal`, `/contact` | Vercel | Edge cache HIT - static. ✓ | Confirms static-build commits deployed. |
| New Sentry issue with first-seen release matching latest deploy | Sentry | Regression. | Consider Vercel rollback. |
| Supabase egress > 4GB/month on free tier (or 200GB on Pro) | Supabase usage dashboard | Throttling produces symptoms identical to pool exhaustion. | Cache hot reads, paginate, or upgrade tier. |

---

## Pool exhaustion diagnostic protocol

When you see clustered `Vercel Runtime Timeout` 504s across unrelated paths AND Sentry silent for the same window, **don't EXPLAIN ANALYZE first**. The DB-side query is almost never the bottleneck on this app — the connection-acquisition queue is. Run these in order; stop at the first one that indicts a layer.

1. **`pg_stat_statements`** via Supabase MCP — pull mean_exec_time, max_exec_time, calls for the suspect query. If max < 10s comfortably, SQL is exonerated. (Reference: 2026-05-08 storm — search query mean=5.98ms, max=232ms across 449K calls.)
2. **`show max_connections;`** — verify the actual ceiling. Don't trust prior CLAUDE.md memory; the project may have migrated tiers.
3. **`select application_name, count(*) from pg_stat_activity group by application_name`** — see who reserves slots. Storage API + postgrest + pg_cron + mgmt-api typically take 30-40 of the cap, leaving the remainder for app traffic.
4. **`select * from public.wedged_backend_log where terminated_at > now() - interval '30 minutes';`** via Supabase MCP `execute_sql` - the `cleanup-wedged-supavisor-backends` pg_cron audit. Cluster of recent terminations indicates pool saturation. For `EMAXCONN` strings (postgres.js client-side surrender) search Vercel logs `query: "EMAXCONN"`. The Supabase MCP does NOT expose pooler/Supavisor logs directly (`service` accepts only `api | branch-action | postgres | edge-function | auth | storage | realtime`).
5. **Supabase usage dashboard** (`mcp__supabase__get_project`) — if egress is over quota, throttling produces symptoms identical to pool exhaustion. Check before blaming postgres.js.
6. **Vercel logs filtered to a single request_id** — pull one truncated 504, look for ANY upstream error in the body. Usually empty (lambda killed mid-wait).

The capacity math at Pro+Micro:
- `max_connections = 60`
- ~30-40 reserved by Supabase services → ~21 free
- Per-lambda postgres.js `max` × concurrent-active-lambdas must stay ≤ 21
- Currently `max=5` → ~4 concurrent lambdas before saturation

Fix priority when this fires:
1. Cache the hottest endpoint(s) with `unstable_cache` to lift them off the connection budget.
2. Lower `max` per pool before raising compute — counter-intuitive but correct, because max=10 lets 2 lambdas hog and starves the next 8.
3. Bump compute add-on (Pro+Small +$10/mo → 90 conns) — money fix, lasts longer.
4. Consider Vercel function `maxDuration` bump to 60s on Pro Vercel — buys lambdas more wait budget without fixing the underlying issue.

## Additional gotchas

- **`pg_stat_statements.total_time` is cumulative since DB restart, not since the 2h window.** Reading it as "compute spent in the last 2 hours" overstates by orders of magnitude. For windowed analysis, either diff against a prior snapshot or focus on `mean_exec_time` × `calls`-delta rather than the raw `total_time` column.
- **Vercel billing data lags 3-5 days.** A spike caught in the 2h window will not show up in this month's invoice number for days. Don't tell the user "this is costing you $X right now" based on today's invoice line - tell them the projected rate, and note the lag.
- **All MCP timestamps come back UTC; convert to BDT (+6) before presenting.** Already covered in cardinal rule 0; restating because trend tables and "first-seen release" rows are where mixed UTC/BDT bleeds in most often.
- **Sentry's `events grouped by release` query relies on releases actually being created.** If `SENTRY_RELEASE` injection broke on a deploy, all events from that window tag as `unknown`, producing a misleading "regression in release X" reading. Cross-check by listing recent releases via `mcp__sentry__find_releases` before drawing release conclusions.
- **`pg_stat_activity` is a point-in-time snapshot.** A single read can miss a transient pile-up that fired and cleared between calls. For 2h trend claims about slot pressure, rely on `wedged_backend_log` (the pg_cron audit), not a single `pg_stat_activity` sample.

## Ask at forks

If the work hits a real decision (architecture, scope, ticket-shape, user-facing copy/UX), use `AskUserQuestion` with concrete options + a recommendation - even in auto mode. See AGENTS.md "Working norms".

## Self-update on better patterns

If this session surfaces a better way of doing things (a new fork to flag, a tool that should have been preferred, a recurring confusion worth a paragraph), propose the edit to this skill / `AGENTS.md` / `docs/*.md` *in the same session*. Do not stash in per-agent memory. See AGENTS.md "Working norms".
