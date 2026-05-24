---
name: incident
description: >
  Beton Kemon production incident protocol. When 504s spike, the
  site feels slow, or users report errors, this skill walks the
  diagnostic order from docs/database.md (pg_stat_statements -> max
  connections -> pg_stat_activity -> Vercel logs -> Supabase
  egress) one step at a time, executes each query, names the
  verdict, and stops the moment a layer is indicted. Iron law: no
  fix until root cause. Stop the bleed (Vercel rollback) before
  fix-forward. Use when asked to "incident", "site is down",
  "504 spike", "users reporting errors", "production is broken",
  "is the site on fire", or any acute production issue. For "fix
  this bug" non-acute debugging, use /investigate (gstack)
  instead.
---

# /incident - production diagnostic walker

When something is on fire, the failure mode is reaching for a hypothesis instead of evidence. Beton Kemon's `docs/database.md` codifies a specific diagnostic *order* (cheapest, highest-information first); this skill enforces that order.

**Iron law:** do not propose fixes until a layer is indicted by evidence. The first 5 minutes of an incident are diagnostic; the next 5 are mitigation.

## Phase 0 - Triage (60 seconds)

Before any diagnosis, ask one question: **is the site down (HTTP 5xx > 1%) or just slow (p95 latency >2s)?**

Hit the homepage, a known company page, and the search API. Production has bot protection that 429s curl/wget, so use **Playwright** (`mcp__plugin_playwright_playwright__browser_navigate` + `browser_network_requests`) to capture status codes and timings:

- `https://www.betonkemon.com/`
- `https://www.betonkemon.com/en/c/aarong`
- `https://www.betonkemon.com/api/search?q=aarong`

(For the API endpoint specifically, curl sometimes works since it's not the bot-gated public HTML; try it first if Playwright is overkill, but fall back to Playwright on 429.)

If responses are 5xx, this is an outage; if 200 but slow, this is degradation. Both go through the same diagnostic order, but degradation is less time-pressured.

If the user says "deploy from <N> minutes ago broke it", offer immediate **rollback** as the first option:

> "Last deploy was <sha>. Want me to check the Vercel deployment list and recommend a rollback target before we diagnose? Stop bleed first."

The user can say "diagnose first" if they're not sure the deploy caused it.

## Phase 1 - The diagnostic order (run in order; stop when indicted)

### Step 1 - `pg_stat_statements`: is the SQL fast?

This is the cheapest, highest-information first move. If query latency is comfortably under 10s, the SQL is exonerated and you can move on.

Use Supabase MCP `execute_sql`:

```sql
select
  query,
  calls,
  total_exec_time::int as total_ms,
  mean_exec_time::numeric(10,2) as mean_ms,
  max_exec_time::numeric(10,2) as max_ms,
  stddev_exec_time::numeric(10,2) as stddev_ms
from pg_stat_statements
order by total_exec_time desc
limit 20;
```

**Verdict rules:**
- `max_exec_time` for the suspect query well under 10s -> SQL is NOT the bottleneck. Move to step 2.
- `max_exec_time` > 5s on a public-page-shaped query -> SQL is suspect. Likely missing index or query change. Stop here, name the query.
- `mean_exec_time` rising w-o-w on a hot query -> data growth has crossed a tier; look at indexes.

The 2026-05-08 storm exonerated the search query at this step (mean 5.98ms, max 232ms across 449k calls).

### Step 2 - `show max_connections`: what's the ceiling?

Don't trust prior memory. Always verify the actual number in production right now:

```sql
show max_connections;
```

Expected ~60 on Pro+Micro. If it's different (compute upgraded, or different from what `docs/database.md` says), update the doc.

### Step 3 - `pg_stat_activity`: who has the slots?

```sql
select application_name, state, wait_event_type, wait_event, count(*)
from pg_stat_activity
group by application_name, state, wait_event_type, wait_event
order by count desc;
```

**Verdict rules:**
- Lots of rows in `state = active, wait_event = ClientRead` -> **wedged backends.** This is a Supavisor-shaped failure (`docs/database.md`). The `cleanup-wedged-supavisor-backends` cron job should be terminating them every minute; if it isn't, that's your indictment.
- Free pool < 10 of 21 expected -> **pool exhaustion likely.** Cross-check Vercel function timeouts in step 4.
- Many `idle in transaction` -> a transaction is holding slots without doing work; usually an admin batch that hasn't committed. Check the `/admin/queue` UI.

```sql
-- The wedge view directly:
select count(*) from public.wedged_backend_log where terminated_at >= now() - interval '10 minutes';
```

If wedge count is rising, the cleanup job is doing its work but the upstream lambda population is generating wedges faster than it can clean up. That's an indictment of cache regression -> step 4 + 5.

### Step 4 - Vercel logs: which paths timed out?

Use the `/website-logs-10m` skill (multi-source: Vercel + Sentry + Supabase, parallel) or the runtime logs MCP directly. Count timeouts in the last 10 minutes by path:

- If multiple unrelated paths timed out together -> shared-resource issue (pool, throttle, region). Confirms step 3.
- If only one path -> endpoint-specific bug. Look at the diff to that path in the last 24h.
- If `Vercel Runtime Timeout Error` appears with no upstream DB error -> classic pool-exhaustion or wedge symptom (lambda killed mid-wait). Don't treat the timeout text as the cause.

### Step 5 - Supabase egress dashboard: throttling?

<https://supabase.com/dashboard/project/hasfagvyjbchoprrwgaj/reports/database>

Egress overage triggers throttling that **looks identical** to pool exhaustion. Free tier 5GB/mo is easy to blow at >10K visitors/day; Pro lifts to 250GB.

If the chart shows the bandwidth ceiling has been hit, that's the indictment. The fix isn't more pool; it's reducing egress (cache, search rate limit, robots.txt).

## Phase 2 - Indictment summary

Whatever step indicted a layer, name it explicitly:

```
Indictment: cache regression (step 4 + 5)

Evidence:
  - Search query mean still 6ms (SQL exonerated, step 1)
  - Free Supavisor pool 4 of 21 (low, step 3)
  - Vercel function timeouts spiked at <ts>, all on /[locale]/c/[slug]
  - Cache hit rate on /[locale]/c/[slug] dropped from 71% to 14%
  - Last commit touching SSR is <sha>: <message>

Hypothesis: <commit-sha> regressed ISR. Probable cause: middleware
adding cookie / forced-dynamic / revalidatePath miss.
```

**Do NOT propose fixes until this summary is written.** The temptation under pressure is to start typing edits before the diagnostic finishes. Resist.

## Phase 3 - Mitigation

Now and only now, propose action. Order:

1. **Rollback** if a recent deploy is implicated. Vercel "Promote to Production" on the previous deployment is the fastest path. See `docs/operations.md` rollback section.
2. **Disable / scale.** If the indictment is non-deploy (sudden traffic, scraper, wedge storm):
   - Vercel Attack Challenge Mode (free toggle, `tasks/open/025`).
   - Pause admin batches if they're contending.
   - Bump Supabase compute (last resort; cache fix first per ADR-0006).
3. **Apply the targeted fix** once the bleeding has stopped.

## Phase 4 - Postmortem

Write a short postmortem into `tasks/done/incident-<YYYY-MM-DD>-<slug>.md` with:

- Timeline (detection, mitigation, resolution).
- Indictment evidence (cite the queries you ran).
- Root cause (one or two sentences).
- Mitigation applied.
- Permanent fix or follow-up tickets.
- New ADR if a hard rule changed.

Per the user's commit-per-todo memory, commit the postmortem as its own commit.

## Postmortem output (HTML)

After Phase 4's markdown postmortem lands in `tasks/done/`, the skill also produces a shareable HTML postmortem at `out/incident/INC-YYYY-MM-DD-<slug>.html` rendered from the template at `.claude/skills/incident/assets/postmortem-template.html`. A worked example lives at `.claude/skills/incident/assets/example-INC-2026-04-18-pool-exhaustion.html` - open it in a browser to see the target layout.

The template uses `{{PLACEHOLDER}}` markers. Fields split into auto-fill vs operator-fill:

**Auto-filled by the skill from MCP / git data:**

- `{{INCIDENT_ID}}` - `INC-<YYYY-MM-DD>-<slug>` from the date + ticket slug.
- `{{OWNER}}` - from `git config user.name`; falls back to `Anonymous`.
- `{{DETECTED_AT}}`, `{{DURATION}}`, `{{WRITTEN_AT}}` - timestamps from the Vercel / Sentry log queries you already ran in Phase 1. Convert to BDT (UTC+6) before substituting.
- `{{TIMELINE_ITEMS}}` - one `<li class="tl-item">` per moment captured during diagnosis (use `class="key"` for "Impact starts", "First alert", "Root cause identified", `class="win"` for "Mitigated").
- `{{IMPACT_REQUESTS}}` - from Vercel MCP `get_runtime_logs` (count of 5xx on the affected path in the impact window).
- `{{IMPACT_ERROR_RATE}}` - peak from Sentry MCP `search_events` over the same window.
- `{{IMPACT_USERS}}` - approximated from Supabase `execute_sql` against `pg_stat_activity` peak + distinct IPs in Vercel logs.
- `{{IMPACT_DB}}` - whether the pool exhausted (Phase 1 step 3 evidence) and whether the `>=5` hardgate distribution shifted (run a quick count by company; usually unchanged for read-side incidents).
- `{{IMPACT_COST}}` - extra Vercel function invocations vs baseline hour + Supabase egress delta (from the cost dashboard).
- `{{ROOT_CAUSE_CODE}}` - the offending diff / config / SQL captured during indictment.

**Operator fills (the skill prompts for these):**

- `{{INCIDENT_TITLE}}` - one-line headline matching the markdown postmortem.
- `{{SEVERITY_CLASS}}` / `{{SEVERITY_LABEL}}` - `sev1`/`SEV-1`, `sev2`/`SEV-2`, or `sev3`/`SEV-3`.
- `{{TLDR_BODY}}` - the plain-English summary; what broke, how long, how it was mitigated.
- `{{ROOT_CAUSE_PARAGRAPHS}}` - one or two `<p>...</p>` blocks expanding on the named root cause.
- `{{ACTION_ITEMS}}` - one `<li class="ai-row">` per follow-up (add `class="ai-row done"` for the ones already finished during mitigation); severity dot is `high`/`med`/`low`.

The HTML opens standalone in any browser - no fonts, no external CSS, no JS.

## Anti-patterns (specifically called out in `docs/database.md`)

- **"The CLAUDE.md says limit 200."** It said 200 once. The real number is whatever `show max_connections` returns now. **Always verify.**
- **"No EMAXCONN in postgres logs means no pool exhaustion."** EMAXCONN is emitted by Supavisor, not Postgres. The MCP `service` enum doesn't expose pooler logs directly; postgres.js raises EMAXCONN client-side after `connect_timeout`, after Vercel has already killed the function. **Absence of EMAXCONN does not exonerate the pool.**
- **"EXPLAIN ANALYZE on idle DB tells me production performance."** It tells you the plan, not the timing under contention. `pg_stat_statements` is the timing source.
- **"`statement_timeout` in `src/lib/db.ts` protects us."** It doesn't propagate through Supavisor. DB default is the real ceiling.
- **Skipping the diagnostic to "just look at the logs."** Logs are step 4. Earlier steps are cheaper and more informative.
- **Assuming a single `canceling statement due to statement timeout` in postgres logs is the storm.** It's normal background noise (slow ANALYZE, etc.). Storm-time DB errors come in clusters.
- **Restarting the lambda / database before evidence is captured.** Some failure modes (wedged backends, hot data structures) clear on restart and the evidence is lost. Capture queries first.

## Tone

Methodical. Step-by-step. The user is stressed; your value is the calm. Each step has a clear verdict; don't move on without one.

## Gotchas

- **Vercel rollback is "Promote to Production" on the prior deployment, not `git revert`.** The fast path is the Vercel dashboard or `vercel promote <prev-deploy-url>` - reverting in git and waiting for a new build wastes the bleed-stop window.
- **Don't `pg_terminate_backend` wedged Supavisor backends manually.** The `cleanup-wedged-supavisor-backends` `pg_cron` job runs every minute and logs to `public.wedged_backend_log`. Manual termination races the cron and obscures the rate of regeneration, which is the actual signal. Only intervene if the cron is paused.
- **Report measured numbers, not impressions.** "Looks better now" is not a verdict. After rollback or mitigation, re-run step 1 (`pg_stat_statements`) and step 3 (`pg_stat_activity`) and quote the deltas. The user has explicitly asked agents to "be truthful and show me the reality."
- **Don't close the incident on green logs alone.** Dev-verify the affected route in Playwright (homepage, `/c/<popular-slug>`, `/api/search?q=...`) before declaring resolved. Logs going quiet can also mean traffic dropped.
- **Close the Playwright window when triage ends.** Any `browser_navigate` session for status-code capture or post-mitigation verification must end with `browser_close` before the postmortem is written.
- **HTML postmortem is a user-facing surface - same hard rules apply (no em-dash, no emoji, plain hyphens). Timestamps in BDT (UTC+6), not UTC.** The template is wired to BDT; when substituting auto-filled fields from Vercel / Sentry / Supabase (all UTC), convert before writing. Run `grep -nP "[\x{2013}\x{2014}]|[\x{1F300}-\x{1FAFF}]" out/incident/<file>.html` before declaring the file done; an em-dash or emoji that leaks in is a hard-rule violation, not a style nit.
- **Read `.env.local` before assuming env state.** Don't guess Supabase project ref, Vercel project name, or Sentry org - the values are on disk. Guessing wastes incident minutes on a wrong dashboard.

## Related

- `docs/database.md` - the canonical diagnostic protocol this skill enforces
- `docs/operations.md` - rollback procedure, dashboards, where-to-look table
- `.claude/skills/website-logs-10m`, `website-logs-2h` - multi-source log counterparts (Vercel + Sentry + Supabase; used in step 4)
- `.claude/skills/database-entry-cleanup` - if the incident requires a remediation DELETE
- `.claude/skills/costanalysis` - the "cost spike" cousin (related but not interchangeable)
- `tasks/done/d001-cache-fix-may-2026.md` - the canonical worked example
- ADR-0002 (pool max=5), ADR-0005 (revalidate sync), ADR-0006 (Supavisor pooling)

## Ask at forks

If the work hits a real decision (architecture, scope, ticket-shape, user-facing copy/UX), use `AskUserQuestion` with concrete options + a recommendation - even in auto mode. See AGENTS.md "Working norms".

## Self-update on better patterns

If this session surfaces a better way of doing things (a new fork to flag, a tool that should have been preferred, a recurring confusion worth a paragraph), propose the edit to this skill / `AGENTS.md` / `docs/*.md` *in the same session*. Do not stash in per-agent memory. See AGENTS.md "Working norms".
