---
name: website-logs-10m
description: >
  Beton Kemon multi-source live triage over the last 10 minutes. Fans out
  in parallel to Vercel runtime logs, Sentry issues/events, and Supabase
  Postgres + pooler logs. Optimized for "is the site on fire RIGHT NOW",
  post-deploy verification within minutes of push, and reproducing
  user-reported errors with full unredacted stack traces. Trigger with
  /website-logs-10m. For trend / billing analysis use /website-logs-2h.
  (Replaces the older /vercel-logs-10m, which only saw Vercel.)
---

# /website-logs-10m - Beton Kemon Live Production Triage (last 10 minutes, all sources)

You are doing **live triage** across **three production data sources** for **Beton Kemon** (`betonkemon.com`):

1. **Vercel runtime logs** - the request-edge view (status codes, function timeouts, edge cache)
2. **Sentry** - the application-error view (full untruncated stack traces, issue grouping, release tagging by `VERCEL_GIT_COMMIT_SHA`)
3. **Supabase logs** - the data-layer view (slow Postgres queries, Supavisor pool exhaustion, edge function errors)

The goal: **is the site on fire right now, and if so, what's the bullet to fix?** Speed of verdict over breadth of analysis.

**Hardcoded coordinates (never prompt):**
- Vercel `projectId`: `prj_4jL6tTaDtN3uo339kSu3XCvcS8vE`
- Vercel `teamId`: `team_Y4Oz01XhgpTFPDVclHoY5GwL`
- Sentry `organizationSlug`: `betonkemon`
- Sentry `projectSlug`: `javascript-nextjs`
- Supabase `project_id`: `hasfagvyjbchoprrwgaj`

---

## Why three sources

Each source sees something the others can't:

| Source | Strength | Blind spot |
|---|---|---|
| Vercel | Sees every request, status codes, edge cache, function timeouts | Truncates messages; can't see DB-layer cause; doesn't group errors |
| Sentry | Full untruncated stack traces, digest, release tagging by deploy SHA, issue grouping | Only sees what app code threw; misses platform-level 504 (function killed before throw) |
| Supabase | Sees slow queries, pool exhaustion, statement timeouts, egress throttling | Doesn't know which Vercel request caused it |

The Vercel-only predecessor skill **infered** DB pool issues from clustered 504s with no upstream error. This skill **confirms** them by reading the pooler/Postgres logs directly. And **infered** error classes from truncated Vercel messages — Sentry has the full digest.

---

## What this skill is for

| Use this when | Use /website-logs-2h when |
|---|---|
| "Something is broken right now" | "How did we do this hour?" |
| "I just deployed, did it land cleanly?" | "Did the perf fix work over the last cycle?" |
| "User reported an error 30 sec ago, find it" | "What's burning my Vercel budget?" |
| "Site feels slow, what's happening?" | Trend / billing analysis |
| Post-push verification (within 5-15 min of deploy) | Post-deploy verification (30+ min after deploy) |

---

## Cardinal rules - read before starting

0. **ALL TIMESTAMPS IN THE REPORT MUST BE BDT (UTC+6).** Vercel, Supabase, and Sentry MCPs return UTC. Convert every single time you cite (table rows, burst windows, deploy times, "what's working" rows, "since the last deploy" math). Never paste a raw UTC time into the report without converting. Format: `22:21 BDT` (or `2026-05-10 22:21 BDT` if the date matters). If the source matters for an operator who'll cross-check the dashboard, show both: `16:21 UTC (22:21 BDT)`. The user is in Bangladesh and reads dashboards in BDT — raw UTC forces them to do +6 in their head on every line. This is non-negotiable; reports that mix UTC and BDT get sent back.

1. **Vercel `runtime_logs` truncates the message column.** Sentry doesn't. Whenever Vercel shows a truncated `[Error: An error occurred i...`, find the same error in Sentry for the full message before drawing conclusions.

2. **Errors-1m is the canary.** If Vercel returns 100 error rows in 1m, that's >100 errors/min - immediate fire. If 1m is clean but 10m has errors, the issue stopped.

3. **Tie everything to deployment / release ID.** Vercel logs show `Deployment: dpl_xxx`. Sentry events tag `release` with `VERCEL_GIT_COMMIT_SHA`. If both surfaces are showing pre-deploy IDs, the new deploy hasn't rolled out yet.

4. **Supavisor pool exhaustion is invisible to Vercel.** When the lambda gets killed at 10s waiting for a DB slot, Vercel shows `Vercel Runtime Timeout Error` with no upstream cause. Supabase pooler logs show the slot starvation. Always check Supabase when Vercel 504s cluster across unrelated paths.

5. **Don't editorialize.** This skill answers four questions in order:
   1. Is anything on fire?
   2. What is it (which layer: app code / platform timeout / DB)?
   3. When did it start?
   4. What's the bullet to fix?
   No billing analysis, no trend tables. Save that for `/website-logs-2h`.

6. **If the user shared a screenshot in conversation, that's the source of truth.** Confirm or contradict it with logs.

---

## Phase 1 - Fire detection (parallel, ~5 seconds wall time)

Send all of these in **one message** as parallel tool calls.

### Group A: Vercel - tight density (the canary)
1. `mcp__vercel__get_runtime_logs` - `level: ["error", "fatal"]`, `since: "1m"`, `limit: 100`
2. same - `since: "5m"`, `limit: 100`
3. same - `since: "10m"`, `limit: 100`

### Group B: Vercel - status codes (10m)
4. `statusCode: "500"`, `since: "10m"`, `limit: 100`
5. `statusCode: "504"`, `since: "10m"`, `limit: 100` - platform timeouts (function ran past 10s)
6. `statusCode: "429"`, `since: "10m"`, `limit: 50` - rate-limit hits

### Group C: Vercel - hot paths
7. `source: ["serverless"]`, `since: "5m"`, `limit: 100` - what's hitting our functions right now
8. `query: "/api/submit"`, `source: ["serverless"]`, `since: "10m"`, `limit: 30` - submit health (canary for DB writes)
9. `query: "Vercel Runtime Timeout"`, `since: "10m"`, `limit: 50` - the pool-exhaustion shape

### Group D: Sentry - issues firing now
10. `mcp__sentry__search_issues` - `organizationSlug: "betonkemon"`, `projectSlug: "javascript-nextjs"`, `naturalLanguageQuery: "issues seen in the last 10 minutes sorted by last_seen desc"`, `limit: 25`
11. `mcp__sentry__search_events` - `organizationSlug: "betonkemon"`, `projectSlug: "javascript-nextjs"`, `naturalLanguageQuery: "error events in the last 10 minutes grouped by issue"`, `limit: 50`

### Group E: Supabase - data layer health
12. `mcp__supabase__get_logs` - `project_id: "hasfagvyjbchoprrwgaj"`, `service: "postgres"` - slow queries, statement_timeout cancellations, errors
13. `mcp__supabase__execute_sql` - `select date_trunc('minute', terminated_at) bucket, count(*) from public.wedged_backend_log where terminated_at > now() - interval '10 minutes' group by 1 order by 1 desc;` - Supavisor backend kills by minute (the `pg_cron` cleanup-job audit; proxy for slot starvation rate). **NOTE:** the MCP `get_logs` tool does NOT expose pooler/Supavisor logs - `service` only accepts `api | branch-action | postgres | edge-function | auth | storage | realtime`. For `EMAXCONN` strings (postgres.js client-side), search Vercel logs `query: "EMAXCONN"`.
14. `mcp__supabase__get_logs` - `project_id: "hasfagvyjbchoprrwgaj"`, `service: "api"` - PostgREST errors (rare, but covers admin actions)

Send all 14 calls in parallel. Total wall time should be a few seconds.

### Verdict gate after Phase 1

- Vercel errors-1m = 100 OR Sentry shows new high-frequency issue → **🔴 FIRE**, jump to Phase 2.
- Vercel errors-1m = 0 but errors-10m has many, OR Sentry shows trailing-off issue → **🟡 SMOLDER**.
- All sources <5 errors → **🟢 CLEAR**, skip to Phase 4.

---

## Phase 2 - Forensics (only if Phase 1 found something)

### 2a. Identify the dominant error class - cross-reference Vercel ↔ Sentry

From Phase 1:
- Pick the highest-frequency Vercel error pattern
- Pick the highest-frequency Sentry issue
- **They should match.** If Vercel shows 50 truncated `[Error: An error occurred i...` and Sentry shows 0 issues in the same window, the error happens at the platform layer (504 timeout, killed before throwing) - **go to 2d (Supabase forensics)**.

If they match, pull the **full message from Sentry** (which has the digest and stack):
- `mcp__sentry__search_issue_events` with the issue ID from Group D
- Or `mcp__sentry__analyze_issue_with_seer` if the issue is non-obvious and you want AI analysis

Capture:
- The error name / digest (Sentry has this; Vercel truncates)
- The exact path that errored (cross-confirm Vercel + Sentry)
- The release tag (`VERCEL_GIT_COMMIT_SHA`) on the Sentry event
- The timestamp of the latest occurrence

### 2b. Find when it started (Vercel narrowing)

Run progressively narrower windows for the same Vercel query:
- `since: "1m"` - did it just start?
- `since: "3m"`, `"5m"`, `"10m"`, `"30m"`

The first window where the count drops to 0 tells you when it started. Compare to deployment timing.

### 2c. Compare deployment / release IDs

Vercel: every log line has `Deployment: dpl_xxx`. Sentry: every event has `release: <git sha>`. If the dominant error is on a single dpl_id / single release, that deployment is the suspect. If errors span multiple, the bug is environmental.

### 2d. Supabase forensics (when Vercel 504s cluster but Sentry is silent)

This is the pool-exhaustion fingerprint. Run in order:

1. `mcp__supabase__execute_sql` - `select count(*), state, application_name from pg_stat_activity group by 2,3 order by 1 desc;` - who is holding slots right now
2. `mcp__supabase__execute_sql` - `show max_connections;` - the actual ceiling (don't trust memory; project may have migrated tiers)
3. `mcp__supabase__execute_sql` - `select query, calls, mean_exec_time, max_exec_time from pg_stat_statements order by max_exec_time desc limit 10;` - which queries are slow
4. Re-check pool exhaustion via `select * from public.wedged_backend_log where terminated_at > now() - interval '10 minutes';` (the `pg_cron` cleanup-job audit). For `EMAXCONN` from postgres.js, search Vercel logs `query: "EMAXCONN"` - the client throws it server-side, surfaces in serverless logs. The Supabase MCP `get_logs` does not expose pooler logs.

Stop at the first layer that indicts. See "Pool exhaustion diagnostic protocol" in `/website-logs-2h` for the full capacity math.

### 2e. Trace one representative request end-to-end

Pick the most recent error's request ID (visible in Vercel log body):
- `mcp__vercel__get_runtime_logs` - `requestId: "req_xxxxx"`, `limit: 20` - every line for that request
- Cross-check Sentry for an event with the same timestamp / path

This gives you a stack-trace-equivalent view across both surfaces.

### 2f. Cross-reference paths the user actually visits

Vercel queries (last 5m, count 200s vs 500s/504s; >25% error rate = user-impacting):
- `query: "/[locale]/c/"` - company pages
- `query: "/api/search"` - search
- `query: "/api/submit"` - submits

---

## Phase 3 - Verdict

🔴 **FIRE** - errors firing at >10/min in last 1-5m. Real users are seeing 500s right now.
🟡 **SMOLDER** - errors trailing off, or low-rate (<10/min) but persistent.
🟢 **CLEAR** - <5 errors total in 10m across all three sources, or only known noise (e.g. `revalidating cache with key`, bot 404s).

---

## Phase 4 - Report (under ~35 lines)

```markdown
## Website Triage (10m) - [BDT timestamp; convert from UTC]
**Verdict:** 🔴 FIRE / 🟡 SMOLDER / 🟢 CLEAR
**Latest deployment:** Vercel [dpl_xxx] / Sentry release [git sha]
**1m / 5m / 10m Vercel error counts:** [N] / [N] / [N]
**Sentry issues seen in 10m:** [N]
**Supabase pooler slots in use / max:** [N] / [N]

### What's happening

[ONE PARAGRAPH if FIRE/SMOLDER, ONE LINE if CLEAR]

[For FIRE/SMOLDER:]
- **Layer:** app-code (Sentry) / platform-timeout (Vercel 504, no Sentry) / DB (Supabase pooler/Postgres)
- **Error class:** [ERROR_NAME with digest from Sentry, not the truncated Vercel message]
- **Affected paths:** [list from Vercel + Sentry intersection]
- **Started:** [time relative to now]
- **Pattern:** [steady / spiking / trailing off]
- **Tied to release?** [yes - SHA xxx introduced it / no - spans multiple releases]
- **Likely root cause:** [one line]

### Bullet to fix

[ONE specific action with file:line if known. Examples:]
- Edit `src/lib/db.ts:33` - bump `idle_timeout` from 5 to 30
- Revert `bf2007f` - the layout `headers()` removal broke something
- Pause traffic via Vercel rollback to dpl_xxx (last known good)
- Wait 2 min - new deploy still rolling out

### Sample evidence
- Sentry issue: [link or ID, full message]
- Vercel last seen: [timestamp]
- Supabase pooler signal: [EMAXCONN count / pg_stat_activity snapshot if relevant]
- Request ID: req_xxxxx (if traceable)
```

If 🟢 CLEAR:

```markdown
## Website Triage (10m) - [BDT timestamp]
**Verdict:** 🟢 CLEAR
**Latest deployment:** Vercel [dpl_xxx] / Sentry release [git sha]
**Vercel 1m / 5m / 10m errors:** [N] / [N] / [N]
**Sentry issues (10m):** [N]
**Supabase pooler:** healthy ([N]/[max] slots)

No fire. [Top serverless paths in last 5m: /[locale], /[locale]/c/[slug], /api/submit, all 200s]. Submit endpoint healthy ([N] 201s in 10m). No new Sentry issues. DB pool well below cap.
```

---

## What NOT to do

- ❌ Don't write more than ~35 lines for FIRE/SMOLDER, ~10 for CLEAR.
- ❌ Don't draw conclusions from a truncated Vercel message when Sentry has the full one.
- ❌ Don't blame a deploy without checking that the dominant error's release tag matches the new SHA.
- ❌ Don't infer pool exhaustion - confirm it via Supabase pooler logs + `pg_stat_activity`.
- ❌ Don't treat `revalidating cache with key` as a real error - it's noise.
- ❌ Don't pull billing data. That's the 2h skill's job.
- ❌ Don't fix the bug in this skill. Output the bullet, then stop.

---

## Beton Kemon-specific signatures (quick lookup)

| Pattern | Where it shows | Verdict if seen >5/min | Quick fix |
|---|---|---|---|
| `Vercel Runtime Timeout Error` cluster across unrelated paths AND Sentry silent for same window | Vercel only | 🔴 FIRE | DB pool exhaustion. Lambda killed at 10s waiting for Supavisor slot, so no app-level throw → Sentry blind. **Confirm with Supabase pooler logs + `pg_stat_activity`.** See pool exhaustion protocol in `/website-logs-2h`. |
| `(EMAXCONN) max client connections reached` | Vercel + Supabase pooler | 🔴 FIRE | postgres.js client-side error. Function survived past 20s. Lower `max` per pool, raise compute, or cache the hot endpoint. |
| `DYNAMIC_SERVER_USAGE` digest | Sentry (digest field) | 🔴 FIRE | Find dynamic API call in layout/page tree. Check `src/app/[locale]/layout.tsx` first. |
| `ECONNREFUSED` to Supabase | Vercel + Sentry | 🔴 FIRE | Supabase down. Check status page + `mcp__supabase__get_project`. |
| Sentry `An error occurred in the Server Components render` with high frequency | Sentry | 🟡 SMOLDER if low rate, 🔴 FIRE if high | Pull issue from Sentry to get the digest. |
| `revalidating cache with key` at error level | Vercel | 🟢 (ignore) | Normal ISR noise. |
| 404s on `/api/companies`, `/meta.json` | Vercel | 🟢 (ignore) | Bot probes. |
| 429s on `/api/submit` | Vercel | 🟢 (working as designed) | Rate limit firing correctly. |
| 504 cluster on a single deployment ID + matching release in Sentry | Vercel + Sentry | 🔴 FIRE | Deploy regression. Consider Vercel rollback. |
| Supabase Postgres `canceling statement due to statement timeout` cluster | Supabase only | 🟡 INVESTIGATE | One stray cancel = noise. Cluster = a hot query is hitting the 10s DB-side cap. Check `pg_stat_statements`. |

---

## Speed targets

A `/website-logs-10m` run should:
- Complete Phase 1 (parallel fetches across all 3 sources) in under 8 seconds wall time
- Complete Phase 2 only if Phase 1 found something
- Output verdict + bullet in under 60 seconds total

If a 🟢 CLEAR run takes more than 15 seconds, you're over-thinking it.
If a 🔴 FIRE run takes more than 90 seconds, you're not parallelizing the forensics queries.

## Additional gotchas

- **Sentry release tagging lags Vercel deploy by 30-90s.** If a fresh deploy shows no Sentry events for the new SHA, that does not mean the deploy is clean - the release just hasn't propagated yet. Wait, recheck, then conclude.
- **All MCP timestamps come back UTC; convert to BDT (+6) before presenting.** Vercel, Sentry, and Supabase all return UTC. Per AGENTS.md working norms, every cited time in the report must be BDT (e.g. `22:21 BDT`), or paired (`16:21 UTC (22:21 BDT)`) when the operator needs to cross-check a dashboard.
- **A "fixed" verdict needs a measured drop, not just a code change.** Don't report a fire as resolved because you committed a fix - confirm the 1m/5m error counts have dropped before flipping the verdict to CLEAR.
- **Sentry's `analyze_issue_with_seer` is slow (10-30s).** Only call it on non-obvious issues where the digest doesn't tell the story. For a known signature (`DYNAMIC_SERVER_USAGE`, `EMAXCONN`), skip Seer and go straight to the fix.

## Ask at forks

If the work hits a real decision (architecture, scope, ticket-shape, user-facing copy/UX), use `AskUserQuestion` with concrete options + a recommendation - even in auto mode. See AGENTS.md "Working norms".

## Self-update on better patterns

If this session surfaces a better way of doing things (a new fork to flag, a tool that should have been preferred, a recurring confusion worth a paragraph), propose the edit to this skill / `AGENTS.md` / `docs/*.md` *in the same session*. Do not stash in per-agent memory. See AGENTS.md "Working norms".
