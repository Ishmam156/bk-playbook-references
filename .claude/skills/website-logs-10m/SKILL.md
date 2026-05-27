---
name: website-logs-10m
description: >
  Beton Kemon multi-source live triage over the last 10 minutes. Hard auth-gate
  on Vercel, Sentry, and Supabase MCPs - refuses to run if any of the three are
  not authenticated. Produces a visually structured report: one section per
  source (Vercel / Sentry / Supabase), each with a technical layer AND a plain-
  language "explain like I'm 10" layer; then a combined synthesis classifying
  findings as Good / Challenging / Alarming; then an AskUserQuestion to guide
  next steps. Trigger with /website-logs-10m. For trend / billing analysis use
  /website-logs-2h.
---

# /website-logs-10m - Beton Kemon Live Production Triage (last 10 minutes)

You are doing **live triage** across **three production data sources** for **Beton Kemon** (`betonkemon.com`). This skill MUST run as four phases in strict order:

```
Phase 0  → Auth gate. If any MCP is missing, STOP and tell the user.
Phase 1  → Three parallel fetches (one section per source).
Phase 2  → Combined synthesis (Good / Challenging / Alarming).
Phase 3  → AskUserQuestion to guide next steps.
```

Every report block has **two layers**: a technical layer (numbers, paths, file references) AND an "explain like I'm 10" (ELI10) plain-language layer. The user is not always in engineering-brain — the ELI10 layer is non-negotiable.

**Hardcoded coordinates (never prompt):**
- Vercel `projectId`: `<VERCEL_PROJECT_ID>`
- Vercel `teamId`: `<VERCEL_TEAM_ID>`
- Sentry `organizationSlug`: `betonkemon`
- Sentry `projectSlug`: `javascript-nextjs`
- Supabase `project_id`: `<SUPABASE_PROJECT_REF>`

---

## Phase 0 - Auth gate (MANDATORY, FIRST)

**Before any data fetch**, verify all three MCP servers are authenticated. The standalone MCPs (`mcp__sentry__*`, `mcp__vercel__*`, `mcp__supabase__*`) are the user-controlled OAuth versions — these are what this skill uses. Do NOT silently fall back to `mcp__claude_ai_*` variants; if the user is running this skill, they expect their own auth.

Run all three checks in parallel:

1. `ToolSearch` with `query: "select:mcp__sentry__whoami,mcp__vercel__list_teams,mcp__supabase__list_organizations"` — confirms the tool schemas are loadable.
2. If schemas load, call them to confirm the auth is **live** (a token that's expired will load the schema but fail at call time):
   - `mcp__sentry__whoami`
   - `mcp__vercel__list_teams` (limit: 1)
   - `mcp__supabase__list_organizations`

### If ANY of the three fails

**STOP. Do not fetch any logs.** Output exactly this and end the turn:

```markdown
## ⛔ Cannot run /website-logs-10m yet

This skill needs all three production MCPs authenticated. Status:

- Vercel:   [✅ authenticated / ❌ not authenticated]
- Sentry:   [✅ authenticated / ❌ not authenticated]
- Supabase: [✅ authenticated / ❌ not authenticated]

**To fix:** run `/mcp` and authenticate the missing one(s), then re-run `/website-logs-10m`.

**Why this matters (ELI10):** This skill is a three-camera security feed. Vercel is the camera at the front door (who came in, who got an error page). Sentry is the camera inside (what tripped). Supabase is the camera in the vault (was the database struggling). Running with only one or two cameras gives a misleading picture — last time we tried that, we mis-classified a real error as "framework noise" because Sentry was missing. Better to wait 30 seconds for you to authenticate than ship a wrong verdict.
```

Then **stop**. No tool calls, no further sections. The user will re-run after authenticating.

### If all three pass

Briefly state "Auth check passed for Vercel, Sentry, Supabase. Pulling logs now." in one line, then proceed to Phase 1.

---

## Phase 1 - Three parallel fetches, three separate sections

Fan out all of the following in **one message** as parallel tool calls. Then write three separate sections in the report — one per source. Each section follows the same template (see Report Structure below).

### Vercel fetches (request-edge view)
1. `mcp__vercel__get_runtime_logs` - `level: ["error", "fatal"]`, `since: "1m"`, `limit: 100` — the canary
2. same - `since: "5m"`, `limit: 100`
3. same - `since: "10m"`, `limit: 100`
4. `statusCode: "504"`, `since: "10m"`, `limit: 50` — platform timeouts (DB pool exhaustion signal)
5. `statusCode: "500"`, `since: "10m"`, `limit: 50`
6. `statusCode: "429"`, `since: "10m"`, `limit: 50` — rate-limit hits
7. `source: ["serverless"]`, `since: "5m"`, `limit: 100` — what's hitting our functions now (load proxy)
8. `query: "/api/submit"`, `since: "10m"`, `limit: 30` — submit health (DB-write canary)

### Sentry fetches (application-error view)
9. `mcp__sentry__search_issues` - `query: "is:unresolved lastSeen:-10m"`, `sort: "freq"`, `limit: 15`
10. `mcp__sentry__search_events` - `dataset: "errors"`, `query: "level:error"`, `fields: ["issue", "title", "count()"]`, `sort: "-count()"`, `statsPeriod: "10m"`, `limit: 15`
11. `mcp__sentry__search_events` - `dataset: "errors"`, `query: "level:error"`, `fields: ["release", "count()"]`, `sort: "-count()"`, `statsPeriod: "10m"`, `limit: 5` — release tag distribution

### Supabase fetches (data-layer view)
12. `mcp__supabase__get_logs` - `service: "postgres"` — slow queries, statement_timeout cancellations
13. `mcp__supabase__execute_sql` - `select date_trunc('minute', terminated_at) bucket, count(*) from <wedged-log-table> where terminated_at > now() - interval '10 minutes' group by 1 order by 1 desc;` — Supavisor backend kills (the pool-pressure audit)
14. `mcp__supabase__execute_sql` - `select count(*) as conn_count, state, application_name from pg_stat_activity group by 2, 3 order by conn_count desc;` — current slot usage
15. `mcp__supabase__execute_sql` - `show max_connections;` — the actual ceiling (project tier may have migrated)
16. `mcp__supabase__execute_sql` - `select query, calls, round(mean_exec_time::numeric, 2) as mean_ms, round(max_exec_time::numeric, 2) as max_ms from pg_stat_statements where calls > 50 order by mean_exec_time desc limit 5;` — slow-query top-5

Note: Supabase MCP `get_logs` does NOT expose Supavisor/pooler logs (`service` only accepts `api | branch-action | postgres | edge-function | auth | storage | realtime`). For `EMAXCONN` strings (postgres.js client-side surrender), search Vercel logs with `query: "EMAXCONN"`. Already covered by call #1-3 if it fires.

---

## Phase 1 - Report structure (three sections, dual-layer)

Write **three separate sections** in this exact order. Do not merge them. Each section uses the same shape:

### Template for each source section

```markdown
## [Vercel | Sentry | Supabase] says

**One-line ELI10:** [What this source watches, in plain English. 1 sentence.]

**Load right now (technical):**
- [Metric 1: e.g. ~X serverless requests / 5m]
- [Metric 2: e.g. Y% error rate]
- [Metric 3: deployment / release ID]

**Load right now (ELI10):**
[1-2 sentences. "About X people hit the site in the last 5 minutes. Y of them got an error page. The site you're showing them is the version you pushed at HH:MM BDT."]

**What's working ✅ (technical):**
- [Bullet 1 with numbers]
- [Bullet 2]

**What's working ✅ (ELI10):**
[1-2 sentences. Plain language version of the bullets above.]

**What's not working ⚠️ (technical):**
- [Error class | path | rate | first-seen release | sample timestamp BDT]
- [Same for next error]

**What's not working ⚠️ (ELI10):**
[1-2 sentences per error class. "When users go to a company page on iPhone, the page crashes and they see a blank screen. It happened 25 times today."]

**What's just noise 🟡 (technical):**
- [bot 404s, framework console.errors, etc.]

**What's just noise 🟡 (ELI10):**
[1 sentence. "Some robots are poking at fake URLs — not a real problem."]
```

### What each source watches (use these for the "One-line ELI10")

- **Vercel** is the front door. It logs every visitor request and tells us if the page came back fine, slowly, or as an error.
- **Sentry** is the alarm system inside the app. When the code actually crashes, Sentry captures the full crash report — which line, which browser, which user.
- **Supabase** is the database. It tells us whether the data layer is healthy (queries fast, plenty of free connection slots) or struggling.

### Layer rules

- The technical layer is for an engineer cross-checking dashboards. Include paths, file references, BDT timestamps, release SHAs, raw numbers.
- The ELI10 layer is for someone who doesn't know what "DB pool" or "release SHA" means. Use plain English. Use comparisons. Never use jargon without explaining it inline.
- Each layer is independently complete. The user should be able to skim only ELI10 lines and get the gist, OR skim only technical lines and have all the references they need.

### Special handling: when a section has nothing to report

If a source is fully clean over 10m, still write the section — just compress it:

```markdown
## Sentry says

**One-line ELI10:** The alarm system inside the app.
**Status:** ✅ Clean — 0 new errors in the last 10 minutes.
**ELI10:** Nothing has crashed inside the app in the last 10 minutes. Quiet.
```

---

## Phase 2 - Combined synthesis

After the three source sections, write ONE combined synthesis section that classifies everything into three buckets. This is where the user looks first to know what to do.

```markdown
## 🟢 What's good / 🟡 What's challenging / 🔴 What's alarming

### 🟢 What's good
**Technical:**
- [bullet 1, citing source: "Vercel: 0 5xx errors in 10m"]
- [bullet 2]
**ELI10:**
[2-3 sentences. "Almost everyone who visited the site got the page they asked for. The database has tons of room. Nothing has crashed today."]

### 🟡 What's challenging (worth fixing this week, not on fire)
**Technical:**
- [issue | rate | layer | rough fix size]
**ELI10:**
[2-3 sentences explaining WHY it matters even though it's not urgent. "There's a small bug in the search box that throws an error when users navigate away mid-search. It doesn't hurt them — they just don't see the search results — but it spams our error log so we miss the bigger fires."]

### 🔴 What's alarming (look at this today)
**Technical:**
- [issue | rate | layer | suspected root cause | suggested fix file:line if known]
**ELI10:**
[3-4 sentences. What it is, who it affects, how often, why it matters. "Twenty-five users opened a company page on iPhone Chrome and the entire page crashed — they saw a blank screen. This is happening to about 1 in every 30 iPhone users right now. We need to find which line of code is calling itself in a loop. Until that's fixed, every iPhone user is rolling dice on whether the page loads."]

If a bucket is empty, say so explicitly: "🔴 Nothing alarming right now."
```

### Classification rules

| Bucket | Threshold |
|---|---|
| 🟢 Good | Working as intended. Cite specific numbers / files. |
| 🟡 Challenging | <10 events / 10m, OR caught by error boundary, OR known-noise that's mildly costly (search-bar AbortError, sessionStorage SecurityError). Fix when you have an hour. |
| 🔴 Alarming | ≥10 events / 10m of an unhandled error, OR a clustered 504 with no upstream Sentry event (pool exhaustion shape), OR a new error class first-seen on the current deploy release SHA, OR `EMAXCONN`, OR `ECONNREFUSED` to Supabase. Look at it today. |

---

## Phase 3 - AskUserQuestion to guide next steps

**ALWAYS end with `AskUserQuestion`.** Never leave the user staring at the report wondering "okay, what now?". Tailor the options to what was actually found.

Question shape:

- **Question** (always end with `?`): something concrete tied to the report findings. NOT "what do you want to do?" — instead "Want me to dig into the iPhone crash first?" or "Should I open a ticket for the search-bar bug?"
- **Header** (≤12 chars): e.g. `Next step`
- **Options** (2-4): each option must be a concrete action you can take next, with a one-line description of what it commits to. The first option should be the recommended action with `(Recommended)` suffix on the label.

### Option archetypes to draw from

Pick 3 that fit the findings. Always include "Stop here, just wanted the read" as an option.

- **Deep-dive on the alarming class** → "Run Seer on Sentry issue X for code-level root cause"
- **Open a ticket** → "Add to `tasks/open/` via /backlog-triage"
- **Verify in browser** → "Reproduce in Playwright on the affected route"
- **Look at trend over longer window** → "Run /website-logs-2h to see the 2-hour shape"
- **Patch the small thing now** → "Edit the file directly (cite path)"
- **Stop here** → "No further action, just wanted the read"

### Example AskUserQuestion call

If the report flagged a 🔴 RangeError on iPhone and a 🟡 SearchBar AbortError:

```
AskUserQuestion({
  questions: [{
    question: "The alarming finding is the iPhone RangeError on company pages (25 events / 48h). What do you want to do next?",
    header: "Next step",
    multiSelect: false,
    options: [
      { label: "Run Seer on the RangeError (Recommended)", description: "Sentry's AI analyzes the stack trace and proposes a code fix. Takes 2-5 min. Won't edit code." },
      { label: "Open a ticket for both findings", description: "Adds RangeError + SearchBar AbortError to tasks/open/ so they're tracked." },
      { label: "Patch the SearchBar AbortError now", description: "Edit src/components/SearchBar.tsx to catch the abort. Small fix, ~5 min." },
      { label: "Stop here, just wanted the read", description: "No further action. I'll come back to it." },
    ],
  }],
})
```

---

## Cardinal rules

0. **Auth gate is non-negotiable.** If any of Vercel/Sentry/Supabase MCP fails the live-auth check in Phase 0, stop. Do not "best-effort" with two sources. The May 2026 audit that mis-classified "Expected the resume..." as a real error happened specifically because Sentry was missing — we will not repeat that.

1. **All timestamps in BDT (UTC+6).** Vercel, Sentry, Supabase all return UTC. Convert every cited time. Format: `22:21 BDT` or `2026-05-25 05:35 BDT` if date matters. If the operator will cross-check a dashboard, show both: `16:21 UTC (22:21 BDT)`. Reports mixing UTC and BDT get sent back.

2. **Dual-layer is non-negotiable.** Every technical block has an ELI10 counterpart. No exceptions, even for "obvious" findings. Reports without ELI10 layers get sent back.

3. **Three separate sections in Phase 1.** Do NOT merge Vercel + Sentry + Supabase into one table. The user explicitly wants to see what each source says independently before the synthesis. Order: Vercel → Sentry → Supabase.

4. **Vercel `runtime_logs` truncates messages, Sentry doesn't.** Whenever a Vercel error appears truncated, cross-reference Sentry by issue title or path before classifying. A class that's loud in Vercel but absent in Sentry is usually framework noise filtered by `beforeSend` — call this out explicitly in the ELI10 ("This shows up in the front-door log but the alarm system inside didn't fire — it's loud but harmless.").

5. **Supavisor pool exhaustion is invisible to Vercel.** When the lambda gets killed at 10s waiting for a DB slot, Vercel shows `Vercel Runtime Timeout Error` with no upstream cause and Sentry stays silent (the app never threw). The fingerprint is: clustered 504s across unrelated paths + Sentry silent + non-zero `<wedged-log-table>` rows. This is the most common 🔴 misclassification — if you see clustered 504s with no Sentry events for the same window, dig into Supabase before concluding "no real errors."

6. **A "fixed" verdict needs a measured drop, not just a code change.** Don't flip a 🔴 to 🟢 because a fix landed — confirm the 1m/5m error counts have dropped in a subsequent run.

7. **End with AskUserQuestion. Always.** Never leave the user without a concrete next step. Auto-mode does not exempt this skill from the AskUserQuestion call; this skill exists to drive decisions, not produce status reports.

---

## Beton Kemon-specific signatures (quick reference)

| Pattern | Where it shows | Bucket | ELI10 |
|---|---|---|---|
| `Vercel Runtime Timeout` cluster across unrelated paths AND Sentry silent | Vercel only | 🔴 Alarming | The database ran out of connection slots. Pages timed out waiting. Users got blank screens. |
| `EMAXCONN` from postgres.js | Vercel | 🔴 Alarming | Same as above — database said "no more connections, sorry." |
| `RangeError: Maximum call stack size exceeded` on a single browser version | Sentry | 🔴 Alarming if recurring | The page got stuck in a code loop on that specific browser and crashed entirely. |
| `DYNAMIC_SERVER_USAGE` digest | Sentry | 🔴 Alarming | A page that was supposed to be cached tried to read a per-user value. The cache is now broken. |
| `AbortError: signal is aborted` from `src/components/SearchBar.tsx` | Sentry (low rate) | 🟡 Challenging | When a user clicks away mid-search, the search request gets cancelled and the cancellation is logged as an error. Harmless to users but spams our error log. |
| `SecurityError: sessionStorage access denied`, `handled=yes` | Sentry (caught) | 🟡 Challenging | Some browsers (usually Facebook in-app browser) block storage. The app catches it gracefully but it still shows up in the log. |
| `Error: Expected the resume...` in Vercel only, not in Sentry | Vercel only | 🟡 Noise | Next.js framework chatter, not a real crash. Filtered out by our error reporter on purpose. |
| 404 on `/api/companies`, `/meta.json` | Vercel | 🟡 Noise | Bots looking for endpoints we don't have. Ignore. |
| 504 cluster on a single deployment SHA + matching Sentry release | Vercel + Sentry | 🔴 Alarming | The last deploy broke something. Consider rollback. |
| `canceling statement due to statement timeout` in Postgres | Supabase only | 🟡 Investigate if clustered | A query took too long and got killed. One = noise. Cluster = a hot query needs optimizing. |
| `ECONNREFUSED` to Supabase | Vercel + Sentry | 🔴 Alarming | The database itself is unreachable. Check Supabase status page. |
| Bursty same-second 429s on `/api/search*` | Vercel | 🟡 Challenging | The rate limiter is firing on the same client multiple times per second — usually a frontend bug double-firing requests. |

---

## What NOT to do

- ❌ Do not run Phase 1 if Phase 0 fails. Stop and ask the user to authenticate.
- ❌ Do not silently fall back to `mcp__claude_ai_*` MCPs when standalone ones are missing. They're different connections; the user expects the one they authenticated.
- ❌ Do not collapse the three source sections into one table.
- ❌ Do not skip the ELI10 layer for "obvious" findings. The user explicitly asked for both layers everywhere.
- ❌ Do not skip the AskUserQuestion at the end, even on a 🟢 all-clear report.
- ❌ Do not draw conclusions from a truncated Vercel message when Sentry has the full one.
- ❌ Do not blame a deploy without checking that the dominant error's release tag matches the new SHA.
- ❌ Do not infer pool exhaustion — confirm it via `<wedged-log-table>` and `pg_stat_activity`.
- ❌ Do not pull billing or trend data. That's the 2h skill's job.
- ❌ Do not fix the bug in this skill. Surface the finding, then ask via AskUserQuestion whether to act.

---

## Self-update on better patterns

If a run surfaces a better way of doing things — a recurring confusion that the ELI10 layer didn't catch, a new signature to add to the table, a fork worth asking about — propose the edit to this skill **in the same session**. Do not stash in per-agent memory. See AGENTS.md "Working norms".
