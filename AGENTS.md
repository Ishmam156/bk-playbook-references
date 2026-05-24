# Beton Kemon - Agent Spec

Crowdsourced anonymous salary transparency for Bangladesh. Bangla-default. Live at <https://www.betonkemon.com>.

This file is the contract Claude Code reads first. Keep it lean. Detail lives in `docs/`. `CLAUDE.md` is a symlink to this file.

## Read before changing code

| If you are touching... | Read first |
| --- | --- |
| anything | this file, top to bottom |
| **a user-visible flow (any of)** | the corresponding `docs/product/*.md` |
| landing / browse / filters / sort | `docs/product/home.md` |
| search bar, `/api/search`, role suggest | `docs/product/search.md` |
| `/c/[slug]` company page, hero, role cards, benefit cells | `docs/product/company-page.md` |
| `/[locale]/departments` tab + `/[locale]/departments/[slug]` drilldown, dept MVs | `docs/product/departments.md` |
| `/[locale]/roles` tab + `/[locale]/roles/[slug]` drilldown, role MV, `/api/search/roles` | `docs/product/roles.md` |
| `/contribute` flow, `/api/submit`, auto-approve | `docs/product/contribute.md` |
| `/onboarding`, locale gate, trust copy | `docs/product/onboarding.md` |
| `/admin/*`, moderation, reports, mappings, branding | `docs/product/admin-moderation.md` |
| social media posts / replies / press answers / image announcements (any external comms) | `docs/social/MASTER.md` (the brand voice canon) |
| picking up a ticket end-to-end | `docs/sdlc.md` |
| Next.js routes / caching / `cookies()` / `headers()` | `docs/next-js-16.md` |
| `src/lib/db.ts`, query latency, 504s, pool config | `docs/database.md` |
| schema, migrations, aggregates | `docs/architecture.md` + `supabase/migrations/` |
| design tokens, mockups, icons | `docs/design-system.md` (+ `docs/conventions.md`) |
| **writing ANY HTML artifact under `out/**`** (audit, snapshot, log report, cost reconciliation, design review, postmortem, ideation demo) | `docs/report-styling.md` - mandatory `--bk-*` token preamble; brand hard rules apply to artifacts, not just production |
| copy, design rules, i18n catalogs | `docs/conventions.md` |
| writing tests / coverage gaps / how to run | `docs/testing.md` |
| CI pipeline, deploy + rollback, dashboards | `docs/ci-cd.md`, `docs/operations.md` |
| `src/app/admin/*` server actions specifically | `docs/admin.md` |
| library / framework / SDK / CLI usage (Next.js, Tailwind v4, `postgres.js`, `next-intl`, `next-themes`, Resend, Turnstile, Vitest, Playwright, Supabase JS, Vercel CLI, etc.) | **Context7 MCP** (`mcp__context7__resolve-library-id` -> `mcp__context7__query-docs`) before training-data answers |
| GitHub issues / PRs / code search / commit history / branch / release ops | **GitHub MCP** (`mcp__github__*`) instead of `gh` shell where the MCP tool exists |
| **why** any rule below exists | `docs/adr/` |

## Coding baseline (general agent behavior)

These are *generic* defaults for LLM coding pitfalls. Project-specific rules below override on conflict.

1. **Surface assumptions before coding.** Before non-trivial implementation, state in one line what you're assuming about input shape, scope, and intent. If two reasonable interpretations exist, say both and pick one with a one-line reason - don't pick silently. If something is genuinely unclear and matters, name the confusion. This is the *in-line* layer beneath the "Ask at forks" working norm: real forks (architecture, scope, UX copy) still get a proper `AskUserQuestion` round-trip; smaller "I think you mean X" moments just get surfaced in the response.
2. **Self-check for overcomplication.** Before finishing, ask: would a senior engineer call this overcomplicated? If yes, simplify. Concrete tells: a strategy/factory/registry for one use site, a `Manager` class wrapping one function, a config object with one knob, a 200-line implementation where 50 would do. (The system prompt's "don't add features/abstractions/error-handling beyond the task" already covers most of this; this self-check is the catch-net.)
3. **Surgical changes - every changed line traces to the request.** Touch only what the task requires. Don't "improve" adjacent code, comments, formatting, or quote style while you're in the area. Don't add type hints, docstrings, or rewrites the user didn't ask for. Match the file's existing style even if you'd write it differently. If you notice unrelated dead code or a real bug, *mention* it - don't fix it as a drive-by. The test for every hunk in your diff: does this line exist because the user asked for it? If no, revert it. The only exception is orphans your own changes created (an import that's now unused because you deleted its caller) - clean those up.
4. **For non-trivial tasks, write a plan with verify steps.** Turn vague asks into checkable ones: "fix the bug" → "write a failing test that reproduces it, then make it pass"; "add validation" → "tests for invalid inputs, then make them pass"; "refactor X" → "tests green before and after". For multi-step work, state the plan as `step → verify: check` lines before starting. Trivial work (one-line fixes, copy tweaks, obvious renames) skips this and stays terse. This complements the stronger project rules: `/debug` for bugs (no fix without root cause), and "Never close a ticket without dev verification" for tickets (browser-verify before flipping to done).

## Hard rules (non-negotiable; violations are reverted)

1. **No emoji in production UI.** General icon vocabulary is the 14-icon set in `src/components/Icon.tsx`. Per-Department icons live in `src/components/DeptIcon.tsx` (one per `Department` enum value, generated via `scripts/gen-dept-icons.mjs` against Gemini, locked - do not expand outside the enum). New surfaces should reuse Icon first; only mint a new fixed catalog tied 1:1 to an existing taxonomy.
2. **No em dash (`-`) anywhere in user-facing text.** Plain hyphen (`-`) in UI strings, `messages/*.json`, SEO meta, JSON-LD, manifest, JSX text. Code comments are fine. (ADR-0001)
3. **No PII on `salary_entries`.** No IP, no user-agent, no fingerprint, no account, no cross-submission session id. Schema-enforced. (ADR-0003, `SECURITY.md`)
4. **>=5 hardgate** for all reliability aggregates (modal pay day, delay rate). Below threshold show locked-preview empty state with CTA. (ADR-0004)
5. **Idempotency on `/api/submit` + transactional aggregate recompute on approval.** Don't break either. Aggregates recompute inside `sql.begin(tx) { ... }` - never partial. (`src/lib/aggregate.ts`)
6. **`revalidatePath("/admin/queue")` stays SYNCHRONOUS** in admin server actions. Cross-page invalidations (landing tag, per-slug x locale) go in `after()`. (ADR-0005, `src/app/admin/actions.ts`)
7. **`src/lib/db.ts` pool config is load-bearing.** Don't change `max`, `idle_timeout`, or re-enable prepared statements without reading `docs/database.md` AND the relevant fix commits. (ADR-0006)
8. **No Inter, no purple/violet, no `system-ui` as primary.** Latin = Geist, Bangla = Hind Siliguri. CSS tokens are `--bk-*` prefixed; don't invent new colors.
9. **Single canonical form for data.** Company names + role titles in English; numerals in Western digits. Bangla mode localizes UI chrome only. (ADR-0007)
10. **Mobile-first.** Desktop two-pane is currently company-page-only - match that pattern when extending.
11. **`src/lib/format.ts` is client-safe.** No DB / server-only imports.
12. **Skeletons, not spinners.** Any waiting UI (route transition, filter re-fetch, lazy panel) swaps to a shape-matching skeleton built on `src/components/Skeleton.tsx`. No spinners, no `Loading...` text, no opacity dimming as the primary cue. Match the silhouette of the real content so layout doesn't shift. In-page transitions wrap the affected region in a client component owning `useTransition` (reference: `DeptFilterPanel.tsx`). Full guidance in `docs/conventions.md` "Loading states".
13. **Doc-sync rule:** any code change touching routes, schema, hard rules, or pool config updates the relevant `docs/*.md` in the same commit. See `CONTRIBUTING.md`.
14. **HTML artifacts use `--bk-*` tokens, not invented palettes.** Any HTML file written under `out/**` (audits, snapshots, log triage, cost reconciliation, design reviews, postmortems, ideation demos) MUST embed the `:root` `--bk-*` block from `src/app/globals.css` verbatim and use those tokens for every color. No dark "developer-tool" palettes, no Inter, no leading `system-ui`, no purple/violet, no em dashes, no emoji - the brand hard rules (1, 2, 8) apply to artifacts the user opens, not just production routes. Full preamble + pre-flight checklist in `docs/report-styling.md`. Canonical examples: `out/spam-audit/2026-05-13-pending-triage.html`, `.claude/skills/incident/assets/postmortem-template.html`.

## Stack at a glance

Next.js 16 (App Router) - TypeScript - Tailwind v4 - Postgres on Supabase via `postgres.js` + Supavisor - `next-intl` URL-routed i18n (`/bn` default, `/en` opt-in) - Resend magic-link admin auth - Cloudflare Turnstile - Vitest + Playwright - Vercel deploy.

Package manager: **npm**. Do not commit `pnpm-lock.yaml` (Vercel's bundled pnpm fails ERR_INVALID_THIS).

## Repo map

```
src/app/[locale]/...     Public routes - landing, onboarding, contribute, /c/[slug], about, contact, legal
src/app/admin/...        Admin routes (no locale, magic-link gated)
src/app/api/...          submit, search, roles/suggest, admin/*, report, companies, loadtest/*
src/lib/                 db, aggregate, companies, departments, auth, admin-guard, contrast,
                         format, turnstile, env, email, filters, page-size, onboarding, reports,
                         common-roles
src/components/          Icon, SearchBar, CompanyCard, FilterAccordion, SortControl, Pagination,
                         ContributeFAB, Footer, LangToggle, Turnstile, CompaniesSidebar, JsonLd, ...
src/i18n/                next-intl routing + request helpers
supabase/migrations/     SQL migrations (21 files as of 2026-05-08)
messages/                bn.json, en.json
e2e/                     (removed 2026-05-09; Playwright suite pending rebuild per tasks/open/009)
scripts/                 Operational scripts (classify, logos, loadtest harness, seed)
design/                  Mockups + PDF (NOT shipped, reference only)
docs/                    Architecture, database, admin, conventions, operations, ADRs
tasks/                   In-repo ticket system (open/, done/)
.claude/skills/          Project-specific skills (approve-batch, unlock-near-companies, doc-sync, ...)
```

## Key invariants in code (cite by file, not from memory)

- **Pool max = 8** in `src/lib/db.ts` (was 5 pre-2026-05-11 PM bump). Why: at Pro+Small (`max_connections=90`, ~50 free slots after services) max=8 supports ~6 concurrent active lambdas while unblocking 8-wide per-render Promise.all fan-out, which is the binding constraint observed on popular company pages (`/c/bkash`, `/c/brac-bank`, `/c/brain-station-23` were hitting 20-37% timeout at max=5 post-compute-upgrade). `CHUNK` in `src/app/admin/queue/Queue.tsx` must match. Any change requires reading `docs/database.md` and ADR-0002 first.
- **Database default `statement_timeout = 10s`** (set via `alter database postgres set statement_timeout = '10s'`, migration 2026-05-08). The postgres.js startup param does NOT propagate through Supavisor - DB default is the real ceiling.
- **`pg_cron` job `cleanup-wedged-supavisor-backends`** terminates Supavisor backends stuck in `active+ClientRead` >30s. Logs to `public.wedged_backend_log`. Don't disable without replacing.
- **`unstable_cache` with `LANDING_CACHE_TAG`** wraps heavy SSR fetches: `getCompanyBySlug`, `getRolesForCompany`, `getPopularCompanies`, `getDistinctIndustries`, `getApprovedEntryTotal`, `getGlobalStats`. Admin mutations call `updateTag(LANDING_CACHE_TAG)`.
- **Client `CHUNK` size in `src/app/admin/queue/Queue.tsx` must match `max` in `src/lib/db.ts`.** Mismatch silently serializes batches.
- **Live submission counts:** query `salary_entries` directly via Supabase MCP. Snapshot in `docs/operations.md` is for context, not truth.
- **Production has bot protection.** Direct `curl`/`wget`/`fetch` against `https://www.betonkemon.com/*` HTML routes returns 429. Use **Playwright MCP** (`mcp__plugin_playwright_playwright__browser_navigate` + `browser_snapshot` / `browser_network_requests`) for any prod URL probe, smoke test, or cache-header inspection. The `/api/*` JSON routes sometimes pass curl through; fall back to Playwright on 429.
- **`GEMINI_API_KEY` is in `.env.local`** with high credit. For any bulk classification task (spam audits, role/company normalization, fuzzy de-dup, freetext bucketing, content moderation triage) prefer LLM-as-judge over hand-written regex/heuristics - regex under-flags by 5x or more on this dataset. Reference pattern: `gemini-2.5-pro` with `responseMimeType: application/json` for structured ratings, then dump CSV (not JSON) for human review.

## Working norms for agents

These apply to Claude Code on this codebase. They are project-shared, not per-user preferences.

- **Persist learnings here, not in personal memory.** When a session produces durable guidance (a workflow rule, a tool gotcha, a reusable pattern), write it into this file, the relevant skill in `.claude/skills/`, or a `docs/*.md`. Do not stash it in per-agent memory directories - other agents won't see it. Memory is for ephemeral session state, not policy.
- **One commit per completed todo.** When a TaskList item flips to `completed`, stage and commit immediately - don't batch. One commit per finished unit of work; conventional-style subject; include the Co-Authored-By footer the system prompt mandates. The "only commit when explicitly asked" default is satisfied by this standing instruction.
- **Screenshots and scratch artifacts go in `.playwright-mcp/` (gitignored), never the repo root.** When `browser_take_screenshot` needs a `filename`, pass a path under `.playwright-mcp/` (e.g. `.playwright-mcp/role-spacing.png`). Same rule for any scratch PNG/HTML/JSON you generate for your own verification - if it's not a deliverable the user asked for, it does not belong at the repo root or under `out/`. Check `git status` before reporting done; if a stray artifact landed in tracked space, `git rm -f` it before commit.
- **Close the Playwright browser when you're done with it.** Any session that opens a Chromium window via `mcp__plugin_playwright_playwright__browser_navigate` (verification screenshots, snapshot checks, dogfooding a flow) MUST end with `mcp__plugin_playwright_playwright__browser_close` before reporting the task complete - leaving the window open clutters the user's desktop. The only exception is when the user explicitly asks you to keep it open for them to inspect.
- **Clean up E2E test data after every Playwright run.** Currently the suite is removed (rebuild tracked in `tasks/open/009-expand-e2e-coverage.md`); this rule applies to any ad-hoc Playwright that hits the live Supabase project: execute `delete from companies where name like 'E2E Co%';` via Supabase MCP without asking first, and mention the cleanup in the end-of-task summary so it's visible.
- **Show timestamps in BDT (UTC+6).** The user is in Bangladesh; raw UTC adds friction. Vercel/Supabase MCPs return UTC - convert before presenting. Format: `11:41 BDT` or `2026-05-08 11:41 BDT`. If the source matters, show both: `05:41:09 UTC (11:41:09 BDT)`.
- **Sunday is the first day of the work week, NOT Monday.** Bangladesh's work week is Sunday-Thursday. Any user-facing call-to-action with a temporal hook says "Sunday morning," "this Sunday," "next Sunday's update" - never "Monday." Applies to LinkedIn / FB / IBA / Messenger / GYC drafts, in-app copy, email subject lines, and any internal-tool reminder visible to the BD audience. The rule is also restated in `docs/social/MASTER.md` Section 10 ("Always" block); a stale Monday reference is a hard-rule violation, not a polish item. Internal-only calendar / admin / analytics work can still use ISO weeks.
- **Bangla in chat = Banglish (Latin script); Bangla in files = Bangla script.** The user's terminal renders Bangla script as broken / overlapping glyphs, so any Bangla string that needs to be read in chat must be transliterated to Banglish (Latin script, phonetic). Examples: `সবচেয়ে বেশি এন্ট্রি` -> `shobcheye beshi entry`, `সর্বাধিক রিপোর্টেড` -> `shorbadhik reported`, `সবচেয়ে বেশি তথ্য` -> `shobcheye beshi tottho`, `সবচেয়ে বেশি শেয়ার` -> `shobcheye beshi share`. When proposing an i18n string in chat, show the Banglish gloss inline and (optionally) name the actual Bangla string that will land in `messages/bn.json`, e.g. `shobcheye beshi entry (bn.json: proper Bangla script)`. The file edit itself stays proper Bangla script - never commit Banglish to `messages/bn.json` or any user-facing surface.
- **Output review tables as CSV (or short markdown), not JSON.** When a script produces tabular data for the user to scan (audit dumps, classification batches, before/after diffs), default to CSV. JSON is for script-to-script handoff, not human reading.
- **Don't surrender at the first wall.** If a tool fails (rate limit, auth, weak signal), enumerate alternatives before reporting blocked: check `.env.local` for keys, scan the loaded MCP tool list, try Playwright when curl 429s, look for indirect side-effect triggers when an endpoint requires auth you don't have. "First tool failed" ≠ "task is blocked."
- **Ask at forks; do not silently pick.** When a task hits a real decision point - architecture choice (path-based vs query-param routes, MV vs cached query), scope boundary (split a PR or land it whole), copy or UX call that will ship to users, ticket-shape change (rename, split, merge, reprioritize), or any branch where reasonable engineers would disagree - stop and use the `AskUserQuestion` tool with concrete options, a one-line description per option (state what each commits you to, not just a label), and a recommendation marked `(Recommended)`. Do this even in auto mode: auto mode minimizes interruptions for *routine* execution, not for forks. The cost of asking is one round-trip; the cost of guessing wrong is rework or a quiet ADR violation. Skip for trivial choices (variable names, log levels, internal helper shape) - this rule is for choices a human would want a say in.
- **Self-update on better patterns.** When a session surfaces a better way of doing things - a new class of fork worth flagging, a tool that should have been preferred, a recurring confusion that a paragraph in a doc would prevent, a skill that should have triggered but didn't - propose the edit to `AGENTS.md`, the relevant `.claude/skills/*/SKILL.md`, or `docs/*.md` *in the same session*, before the lesson decays. Do not stash these in per-agent memory (it does not survive across agents/projects). Concrete trigger: at the end of any non-trivial task, ask yourself "did I do something this session that the *next* run would benefit from knowing?" If yes, write it down. If unsure, ask the user with one short `AskUserQuestion`. The norm above ("Persist learnings here, not in personal memory") is the *what*; this norm is the *when* - actively propose, do not wait to be asked.
- **Never close a ticket without dev verification.** "Typecheck passes + lint passes + commit landed" is NOT verification — those tools tell you the code compiles, not that the feature works. Before flipping any ticket to `done` (or `git mv`-ing into `tasks/done/`), you MUST: (1) start `npm run dev` (background), (2) navigate to the affected route(s) via Playwright MCP (`mcp__plugin_playwright_playwright__browser_navigate` + `browser_snapshot`), (3) exercise each acceptance criterion from the ticket and confirm visually, (4) check `browser_console_messages` for runtime errors, (5) only then close. If the change is server-only (a script, a migration, a DB helper with no UI surface), run the script/query and show the output. "I wrote the code that should do X" is not the same as "X works." This rule applies even to small/low-priority tickets — the cost of spinning up dev is two minutes; the cost of a silently broken close is the user finding it on prod.
- **Audit AI-generated visuals before shipping.** Any AI-generated image, video, or visual mockup (nano-banana, Gemini image, Pretext-HTML render, etc.) MUST go through this checklist before being delivered to the user: (1) `Read` the output file at full size, (2) walk every spec bullet and mark pass/fail (headline copy, line breaks, brand colors, NO emoji, NO em-dashes, hardware artifacts like double notches/duplicate phones, text legibility), (3) verify dimensions match production target at 2x retina (so 1080x1080 → 2160+ source, 1200x627 → 2400+ source; AI tools default to 1K which is NOT retina-ready - always pass `--size 4K` for production assets), (4) regenerate or post-process (sips crop/resize) if any check fails, (5) only then declare done. "Ship-and-caveat" ("there's a small artifact but it's fine") is not acceptable - the user pattern-matches AI artifacts as sloppy, exactly what defeats the point of a launch asset. When in doubt, regenerate; it costs minutes, not credibility.
- **WebKit-verify any route-level change under `src/app/[locale]/`.** Chrome-only verification is not sufficient. WebKit (Safari, iOS Safari, FB iOS in-app browser) implements DOM `insertBefore` strictly per the WHATWG spec - throws `HierarchyRequestError` when the inserted node is an inclusive ancestor of the parent. Chromium silently no-ops the same case (out-of-spec) and the page renders. The May 2026 `/[locale]/roles*` + `/[locale]/departments/[deptSlug]` blank-Safari incident shipped clean in Chromium and rotted in production for 8 days because nobody opened those routes in real WebKit. Trigger: any change to `page.tsx`, `loading.tsx`, `layout.tsx`, Suspense boundary placement, `async` server-component shape, or `'use cache'` placement under `src/app/[locale]/*`. Before push: `npm run build && npm start` (background), then `node scripts/webkit-smoke.mjs http://localhost:3000 webkit` - expects 18/20 routes healthy (the 2 `/contribute` failures are pre-existing Cloudflare Turnstile dev-key errors). Pass `chromium` instead of `webkit` to cross-check. Reading from prod: `node scripts/webkit-smoke.mjs https://www.betonkemon.com webkit` after a deploy. See `tasks/done/070-...md` for the diagnosis pattern that prompted this norm.
- **Sentry `level=warning` events firing at >1/hr from real user IPs are suspect, not background noise.** The May 11 `JAVASCRIPT-NEXTJS-2E` HierarchyRequestError storm was tagged `level=warning` by `auto.browser.global_handler` and dismissed as cosmetic. The reality: every event was a full-page render failure on iOS Safari. Triage rule: if a Sentry event class fires >1/hr AND its `user.geo` field shows real Bangladesh IPs (not Dublin/AWS crawlers), open the URL in real WebKit before classifying. The `/website-logs-2h` skill audits this window; do not dismiss its warning-tier output without reproducing in the same engine the user reports. Especially relevant for routes our users hit from Facebook iOS (the FB in-app browser is WebKit; we are ~60% iOS-via-Facebook on user traffic).
- **User reports of "X isn't working" demand browser + device + screenshot before theorizing.** Vague reports ("ei link gula kaaj kortese na", "page won't load") triple the debug time and routinely mislead toward wrong hypotheses. The May 19 Raiyaan report initially read as a routing or auth issue; the screenshot showed Safari's "A problem repeatedly occurred" page, which is a renderer crash, not a 4xx. First reply on any reported breakage: "Which browser + device, and can you screenshot?" before code dives. Most BD traffic is iOS Safari (via Facebook in-app browser); a "doesn't work" report without browser context is roughly 60% iOS-Safari-specific. Once you have the screenshot, reproduce in `scripts/webkit-smoke.mjs` before forming hypotheses.
- **Prefer HTML+Playwright for any asset showing app UI; use AI vision as the QA gate.** When a launch graphic, social card, or marketing image needs to show actual product UI (phone mockups of the real tabs, screenshots of real flows), do NOT have an AI image model render the UI - the model drifts between renders, gets fonts wrong, and produces inconsistent details across image variants. Instead: build the asset as an HTML file that uses the actual `--bk-*` tokens (lift markup from `out/ideate/*.html` demos when possible), render with Playwright at `deviceScaleFactor: 4` for 4x retina output, screenshot. Reference pipeline lives in `out/social/_builder-{square,landscape}.html` + `_render.mjs`. The HTML stays editable, brand-consistent, reproducible, and zero AI artifacts. AI image generation is still the right tool for backgrounds, illustrations, photographic textures - things without a canonical UI source. Regardless of how the asset is produced, run an independent AI vision QA pass before delivery: feed the output PNG + the spec checklist to `gemini-2.5-pro` (reference: `out/social/_vision-qa.py`) and require an explicit `overall: PASS` with every numbered check passing. This catches issues the eye glosses over (double bezels, missing home bar, font fallbacks) and proves the asset is up to spec rather than relying on the agent's self-report.

## MCP tools (prefer over shell where one exists)

The harness loads several MCP servers. When a task fits one, use it instead of `curl`, `gh`, training-data recall, or hand-rolled SQL.

- **Supabase** (`mcp__supabase__*`) - project: **`betonkemon`**, ref `hasfagvyjbchoprrwgaj`, region `ap-northeast-1`. Pass `project_id: "hasfagvyjbchoprrwgaj"` on every call. Same org has `shikho-ai-os`, `korea-tourbuddy`, `avds-student` - never touch those. Default for: live entry counts, schema inspection (`list_tables`), advisors / log triage (`get_advisors`, `get_logs`), migrations (`apply_migration`), one-off SQL (`execute_sql`). Destructive mutations (`DELETE`/`UPDATE` on `salary_entries`/`roles`/`companies`) go through the `database-entry-cleanup` skill, never raw `execute_sql`.
- **Context7** (`mcp__context7__resolve-library-id` then `mcp__context7__query-docs`) - **default** for any library/framework/SDK/CLI question (Next.js 16 caching APIs, Tailwind v4 tokens, `postgres.js` options, `next-intl` routing, `next-themes`, Resend, Turnstile, Vitest, Playwright, Supabase JS, Vercel CLI, `gh` CLI). Use even when the answer feels obvious - training data lags real releases. Skip for refactors, project business logic, or general programming concepts.
- **GitHub** (`mcp__github__*`) - default for issues, PRs, code search across the org, commit/release lookup, branch ops, PR comments / review threads. Use instead of `gh` shell where the MCP tool covers it. Still use `gh` from Bash for `gh pr create` (HEREDOC body) and other flows the system prompt's PR-creation protocol scripts. When opening a PR: call `mcp__github__update_pull_request` to attach labels (`auto-merge` if CI is green and the diff is small, `needs-review` otherwise) and set the milestone if one is open; after push, call `mcp__github__list_commits` on the new branch to verify the remote head matches local. If the diff closes a `tasks/open/*.md` ticket with a linked GitHub issue, include `Closes #N` and verify the issue exists with `mcp__github__issue_read` first - never invent issue numbers. When reviewing a PR: call `mcp__github__pull_request_read` first and `mcp__github__search_issues` for related open issues touching the same files, follow up on existing threads with `mcp__github__add_reply_to_pull_request_comment` rather than new top-level comments, and post review verdicts via `mcp__github__pull_request_review_write` so threaded comments stay grouped.
- **Sentry** (`mcp__sentry__*`) - issue triage with full stack traces, release tagging, Seer analysis. Used heavily by `incident`, `website-logs-10m`, `website-logs-2h`, and `debug` skills.
- **Vercel** (`mcp__vercel__*`) - deployment status, runtime logs, build logs. Used by the same incident/log skills above and by `costanalysis`.
- **Playwright** (`mcp__plugin_playwright_playwright__*`) - production probes (the site 429s on `curl`), QA flows, screenshot evidence. See the bot-protection note in "Key invariants" above.

Tool-failure ladder before reporting blocked: (1) check `.env.local` for missing keys, (2) try the MCP equivalent of the failing shell command, (3) fall back to Playwright when HTTP probes 429, (4) for indirect access (e.g. an endpoint requires admin auth) trigger the side effect via the relevant action and observe via Sentry/Supabase logs.

## Common commands

```bash
npm run dev           # next dev
npm run build         # next build
npm run lint          # eslint
npm test              # vitest
npm run classify      # offline Gemini role->department classifier
```

## Tickets + roadmap

- **Backlog:** `tasks/open/`, one `.md` per ticket with YAML frontmatter. Index: `tasks/README.md`.
- **Triage:** invoke the `backlog-triage` skill. New ideas go into `tasks/open/` directly.
- **Done:** `git mv` from `tasks/open/` to `tasks/done/`. The diff is the audit log.

## Skills (project-local, `.claude/skills/`)

Full reference + when-to-use table: `docs/skills.md` (grouped by the 9-category Thariq taxonomy). Authoring rules also live there - folder-not-file, description-as-trigger, Gotchas section required, progressive disclosure. Quick list:

- **Library & API Reference:** _(empty - candidate: `gemini-batch`)_
- **Product Verification:** `ticketaction` _(candidates: `contribute-smoke`, `seo-audit`)_
- **Data Fetching & Analysis:** `website-logs-10m`, `website-logs-2h`, `costanalysis`, `data-snapshot`, `spam-audit` _(candidate: `company-merge`)_
- **Business Process & Team Automation:** `ideate`, `backlog-triage`, `todo`, `approve-batch`, `unlock-near-companies`, `runsync`, `social-media-create-post`, `social-media-response`, `social-media-create-image`, `answer-interview-questions` _(candidate: `industry-classify`)_
- **Code Scaffolding & Templates:** _(empty for code surfaces - candidate: `gemini-batch`)_
- **Code Quality & Review:** `doc-sync` (plus four thin graph-MCP wrappers slated for consolidation: `debug-issue`, `explore-codebase`, `refactor-safely`, `review-changes`)
- **CI/CD & Deployment:** _(empty - candidates: `migration-deploy`, `pre-deploy-verify`)_
- **Runbooks:** `incident`, `debug`
- **Infrastructure Operations:** `database-entry-cleanup` _(candidates: `logo-sync`, `audit-invariants`)_

## Out of scope for v1

Feed tab, comp calculator, algorithmic Popular ordering, user accounts, mobile native, runtime-tweakable design tokens, Bangla numerals in UI, Bangla company names. See `tasks/` for v2 parking lot.

## What humans still need to do

`docs/HUMAN-INTERVENTIONS.md`. Read before asking the maintainer for input - it lists what's already pending on their plate.

<!-- code-review-graph MCP tools -->
## MCP Tools: code-review-graph

**IMPORTANT: This project has a knowledge graph. ALWAYS use the
code-review-graph MCP tools BEFORE using Grep/Glob/Read to explore
the codebase.** The graph is faster, cheaper (fewer tokens), and gives
you structural context (callers, dependents, test coverage) that file
scanning cannot.

### When to use graph tools FIRST

- **Exploring code**: `semantic_search_nodes` or `query_graph` instead of Grep
- **Understanding impact**: `get_impact_radius` instead of manually tracing imports
- **Code review**: `detect_changes` + `get_review_context` instead of reading entire files
- **Finding relationships**: `query_graph` with callers_of/callees_of/imports_of/tests_for
- **Architecture questions**: `get_architecture_overview` + `list_communities`

Fall back to Grep/Glob/Read **only** when the graph doesn't cover what you need.

### Key Tools

| Tool | Use when |
| ------ | ---------- |
| `detect_changes` | Reviewing code changes — gives risk-scored analysis |
| `get_review_context` | Need source snippets for review — token-efficient |
| `get_impact_radius` | Understanding blast radius of a change |
| `get_affected_flows` | Finding which execution paths are impacted |
| `query_graph` | Tracing callers, callees, imports, tests, dependencies |
| `semantic_search_nodes` | Finding functions/classes by name or keyword |
| `get_architecture_overview` | Understanding high-level codebase structure |
| `refactor_tool` | Planning renames, finding dead code |

### Workflow

1. The graph auto-updates on file changes (via hooks).
2. Use `detect_changes` for code review.
3. Use `get_affected_flows` to understand impact.
4. Use `query_graph` pattern="tests_for" to check coverage.
