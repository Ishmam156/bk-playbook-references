---
name: costanalysis
description: >
  Beton Kemon cost monitor. Two modes. `live`: pulls current usage
  from Vercel + Supabase via MCP / dashboards (edge requests,
  function invocations, Active CPU, DB egress, query volume),
  flags anomalies vs the last week, identifies the top-cost code
  paths. `reconcile`: ingest a billing CSV the user attaches
  (Vercel monthly, Supabase monthly, Resend, Anthropic, Sentry)
  and stitch a multi-month trend into out/costanalysis/. Use when
  asked to "check costs", "cost analysis", "are we burning
  money", "Vercel bill", "Supabase usage", "monthly costs",
  "reconcile bill", or "where is the money going". Ad-hoc; wrap
  in /loop or /schedule manually if you want a cadence.
---

# /costanalysis - cost & usage monitor

Beton Kemon runs on Vercel + Supabase + Resend + (now) Sentry + Gemini for offline jobs. Costs are mostly low and predictable, but spikes happen (the 2026-05-08 storm, the post-press traffic surge) and several knobs (cache hit rate, function timeout %, Supabase compute tier) directly drive the bill.

This skill answers "what is going on cost-wise" both in real time and when reconciling a month-end statement.

## Modes

| Mode | When | What it does |
| --- | --- | --- |
| `live` (default) | "check costs", "current usage" | Pulls live numbers from Vercel + Supabase, compares to last week, flags anomalies |
| `reconcile` | User attaches a billing CSV | Parses it, stitches with prior months in `out/costanalysis/`, reports trend deltas |
| `forecast` | User asks "what will next month cost" | Linear projection from the last 30 days of live usage |

If the user's input is ambiguous, default to `live`.

## Mode: live

### Step 1 - Pull current usage

Vercel (no MCP for billing yet, so dashboard URLs):

| Metric | Where |
| --- | --- |
| Edge requests (last 7d, 30d) | <https://vercel.com/ishmam156s-projects/beton-kemon-12345/observability> |
| Function invocations + duration | same |
| Function error % + timeout % | same |
| Active CPU hours | same |
| Fast Data Transfer (egress) | same |
| Cache hit rate per route | <https://vercel.com/ishmam156s-projects/beton-kemon-12345/cache> |

Ask the user to capture screenshots OR paste the headline numbers if MCP can't reach the dashboard. The Vercel MCP `mcp__vercel__list_deployments`, `get_deployment`, `get_runtime_logs` give you traffic shape but not billing.

Supabase via MCP (`project_id: hasfagvyjbchoprrwgaj`):

```sql
-- Connection state
select application_name, count(*) from pg_stat_activity group by application_name;

-- Hottest queries by exec time
select query, calls, total_exec_time::int, mean_exec_time::int, max_exec_time::int
from pg_stat_statements
order by total_exec_time desc limit 20;

-- Egress proxy (rows returned by SELECTs is a coarse signal)
select sum(rows)::bigint as rows_returned, count(*) as calls
from pg_stat_statements where query ilike 'select%';

-- Storage usage
select schemaname, sum(pg_relation_size(schemaname || '.' || tablename))::bigint as bytes
from pg_tables where schemaname = 'public' group by schemaname;
```

Plus: <https://supabase.com/dashboard/project/hasfagvyjbchoprrwgaj/reports/database> for the egress chart (free tier 5GB/mo, Pro 250GB).

Resend, Anthropic (if any), Sentry: pull from each provider's dashboard. The user attaches numbers if MCP doesn't reach.

### Step 2 - Compare to baseline

The baseline is "last week" or "the prior 7-day window". For each metric, compute:

```
metric        7d_now     7d_prior    delta
edge_req      2.1M       1.8M        +17%
fn_timeout %  0.4%       0.3%        +33% (within noise)
active_cpu    8.2h       6.1h        +34%  ! flag
egress        18 GB      14 GB       +29%
cache_hit %   62%        71%         -9pp  ! flag
```

Flag rules:
- **Active CPU up >25% week-over-week** -> something's hitting the lambda harder; check cache.
- **Function timeout % > 1.0%** -> outage signal; cross-check `/website-logs-2h` (multi-source).
- **Cache hit % drops >5pp** -> cache regression; usually middleware or revalidatePath issue. See `docs/database.md`.
- **Egress up >30% w-o-w** -> verify it's not bot scraping; `tasks/open/023` (rate limit) or `tasks/open/025` (Vercel Attack Challenge Mode) become relevant.
- **Supabase free pool < 10 of 21 sustained** -> pool exhaustion is the next storm; bump compute (Pro+Micro -> Pro+Small) before it happens.

### Step 3 - Identify top-cost paths

Cross-reference Vercel function invocations (per route) with `pg_stat_statements`:

```
top function invocations:
  /api/search        449k calls   p50 12ms   - hot, cached, OK
  /[locale]/c/[slug] 102k calls   p50 35ms   - SSR, ISR-cached, OK
  /api/companies      53k calls   p50 18ms   - OK
  /admin/queue        2.1k calls  p50 8.2s   - SLOW (admin batches; expected)
  ...
```

If anything in the top 10 is uncached and per-request shaped (per keystroke, etc.), that's the top knob to turn.

### Step 4 - Report

Concise verdict: **green** (nothing actionable), **yellow** (one or two flags worth watching), **red** (act today).

```
Verdict: yellow

Active CPU up 34% w-o-w (8.2h vs 6.1h).
Cache hit % dropped 9pp on /[locale]/c/[slug] (71% -> 62%).

Likely cause: cache regression. Last commit touching SSR is <sha>.
Next action: re-verify the post-deploy checks in docs/operations.md;
if confirmed, follow ticket 002 protocol.
```

Save the report into `out/costanalysis/live-<timestamp>.md` for trend stitching.

## Mode: reconcile

### Step 1 - Accept a CSV

User attaches a billing CSV (Vercel monthly, Supabase monthly, Resend, Sentry, etc.). Each provider's CSV shape is different; the skill must be tolerant.

Parse the CSV. Identify:
- Provider (heuristic: filename, header columns)
- Billing period start + end
- Line items + amounts
- Total

If the shape is unfamiliar, ask the user to identify the columns rather than guess.

### Step 2 - Stitch with prior months

Look in `out/costanalysis/billing/`. Files there should be named `<provider>-<YYYY-MM>.json` (the skill writes them after parsing). If a row for this period already exists, ask before overwriting (could be a re-export with corrections).

Example layout:

```
out/costanalysis/
  billing/
    vercel-2026-04.json
    vercel-2026-05.json
    supabase-2026-04.json
    supabase-2026-05.json
    resend-2026-05.json
    sentry-2026-05.json
  live-<timestamps>.md
  reconcile-2026-05.md   <- the report this run produces
```

### Step 3 - Trend deltas

For each provider, show:

```
Vercel
  2026-03   $12.40
  2026-04   $14.10   (+14%)
  2026-05   $48.70   (+245% !)  <- the storm month

Supabase
  2026-03   $25.00 (Pro base)
  2026-04   $25.00
  2026-05   $35.00 (Pro+Micro upgrade mid-month)

Total
  2026-03   $37.40
  2026-04   $39.10
  2026-05   $83.70
```

Annotate spikes with what changed (cross-reference `tasks/done/d001-cache-fix-may-2026.md`, deploy timeline from git log).

### Step 4 - Forecast

Linear extrapolation of last 30 days at current rate, plus called-out adjustments (e.g., "if Sentry tier moves Team -> Business, +$30/mo"). Caveat that forecast is a sanity check, not an oracle.

### Step 5 - Suggestions

Top 3 cost levers given the trend:

```
1. Verify cache hit rate (cheap; if regressed, it's the highest leverage).
2. Drop Supabase free-tier overages by upgrading egress budget OR
   reducing /api/search egress (tasks/open/023 rate limit).
3. Lower Sentry tracesSampleRate from 0.1 to 0.05 if budget tightens.
```

## Cost levers (cheapest first)

Codified from `docs/operations.md`:

1. Verify cache hit rate. One regression doubles DB load AND function CPU.
2. Confirm `robots.txt` blocks `/api/*` and has Crawl-delay (`tasks/open/026`).
3. Vercel Attack Challenge Mode + Bot Protection Managed Ruleset (free toggle).
4. `/api/search` + `/api/roles/suggest` rate limit (`tasks/open/023`).
5. Supabase compute Micro -> Small (+$10/mo, lifts free pool ~21 -> ~50 slots) - only after exhausting cache fixes.
6. Sentry sample rate (0.1 -> 0.05 trades signal density for half the events).

## Anti-patterns

- **Treating Vercel logs as a billing source.** Logs tell you what happened, not what it cost.
- **Comparing month-over-month without normalizing for traffic.** A 30% cost spike with 30% more traffic is flat per-visitor; flag only when ratio drifts.
- **Acting on a single-day anomaly.** Always show the 7d + 30d window; one-day spikes are routine.
- **Recommending compute bumps before cache verification.** ADR-0006 path: cache first, compute last.
- **Forgetting Gemini.** `/runsync` runs Gemini; bills usually trivial but worth a line in reconcile.
- **Storing PII from billing CSVs.** Some CSVs include cardholder names / addresses. Strip those before writing to `out/costanalysis/billing/` if present.

## Tone

Numbers + verdict + 2-3 actionable suggestions. The user is checking costs, not learning accounting.

## Gotchas

- **Vercel billing data lags 3-5 days.** Live observability shows real-time invocations and CPU, but the invoice line items reconcile on a delay. A "current month invoice" pulled mid-month will undercount; say so explicitly when reporting.
- **Supabase egress is reported per-day with a ~24h delay.** The egress chart you see today is yesterday's number. For a same-day spike investigation, lean on `pg_stat_statements` row counts as a proxy, not the dashboard chart.
- **Always state the timezone on every number.** Vercel and Supabase dashboards default to UTC; the user reads in BDT (UTC+6). A "spike at 02:00" means very different things depending on which - per AGENTS.md, convert before presenting (e.g., `02:00 UTC (08:00 BDT)`).
- **Don't recommend compute bumps before verifying cache hit rate.** ADR-0006 path: cache first, compute last. A regressed cache hit % is almost always the cheaper fix and the real root cause.
- **Forecasts are sanity checks, not oracles.** Linear extrapolation from 30 days of usage will miss step changes (a press hit, a tier upgrade, an Attack Challenge toggle). Always caveat.
- **Strip PII from billing CSVs before writing to `out/costanalysis/billing/`.** Some provider exports include cardholder name and address. Drop those columns at parse time.

## Related

- `docs/operations.md` - dashboards, cost levers
- `docs/database.md` - pool sizing trade-offs
- `tasks/open/002` - cache verification (the recurring task this skill keeps relevant)
- `tasks/open/023, 025, 026` - cost-flavored tickets
- `tasks/done/d001-cache-fix-may-2026.md` - the storm context for any historical reconcile
- `.claude/skills/website-logs-10m`, `.claude/skills/website-logs-2h` - multi-source log counterparts (Vercel + Sentry + Supabase)

## Ask at forks

If the work hits a real decision (architecture, scope, ticket-shape, user-facing copy/UX), use `AskUserQuestion` with concrete options + a recommendation - even in auto mode. See AGENTS.md "Working norms".

## Self-update on better patterns

If this session surfaces a better way of doing things (a new fork to flag, a tool that should have been preferred, a recurring confusion worth a paragraph), propose the edit to this skill / `AGENTS.md` / `docs/*.md` *in the same session*. Do not stash in per-agent memory. See AGENTS.md "Working norms".
