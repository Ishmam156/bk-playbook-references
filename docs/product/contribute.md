# User journey: Contribute (submission)

**Route:** `/[locale]/contribute`
**Files:** `src/app/[locale]/contribute/page.tsx`, `ContributeFlow.tsx`, `src/app/api/submit/route.ts`, `src/lib/turnstile.ts`, `src/lib/aggregate.ts` (`tryAutoApprove`)
**Migrations of interest:** `20260501183033_init_schema.sql` (salary_entries base), `20260502150000_replace_years_experience_with_role_years.sql`, `20260505120000_add_benefits.sql`, `20260503000000_add_entry_source.sql`

## Purpose

Capture a useful, non-fraudulent salary submission in under two minutes from a non-technical visitor on a phone. Every submission costs the contributor real (perceived) social risk; the flow has to signal trust at every step and ask only for what we'll actually use. As of 2026-05-08 the conversion rate (visitors -> submissions) is ~16%, which is high; the flow is doing its job and we should not regress it lightly.

## The flow at a glance

**8 cards, single-purpose, mobile-first.** Draft is auto-persisted to `localStorage` under `bk_contribute_draft` (`ContributeFlow.tsx:12`); a resume banner offers to restore on return.

```
1. Company   -> 2. Role    -> 3. Time period -> 4. Salary
                                                  |
                                                  v
8. Review    <- 7. Benefits (optional) <- 6. Delays <- 5. Pay day
   |
   v Submit
```

Edit-from-review: tapping a row's "Edit" on the review card jumps to that step with `editingFromReview=true`; on completion, the user returns to review rather than advancing. (`ContributeFlow.tsx:486`, `:244-247`).

## Step 1 - Company

**Component:** `CompanyStep` (`ContributeFlow.tsx:668-735`).

**Captures (required):**
- `company.id` (UUID) **OR** `company.name` (string). At least one must be non-null/non-empty (server check at `route.ts:125-128`).

**UX:**
- Search input -> hits `/api/search?q=` with 200ms debounce.
- Existing matches show logo + industry pill + entry count.
- "+ Add new company" path collects an English name and continues.
- URL parameter `?company=<slug>` pre-fills via `/api/companies?slug=` lookup on hydration (`:219-223`). Used by the "Add first salary" CTA on the search miss state and the company page locked previews.

**Server-side resolution** (`route.ts:160-186`):

1. If `company_id` is provided, fetch its slug.
2. Else if `company_name` is provided:
   - Look up by `lower(name)` first (case-insensitive match against existing canonical).
   - If no hit, create with `slugify(name)` and `ON CONFLICT (slug) DO UPDATE` to handle slug collisions atomically.
3. Slugify rules (`route.ts:67-74`): lowercase, strip non-alphanumeric, max 64 chars. "HSBC Bangladesh & Co." -> `hsbc-bangladesh-co`.

**Duplicate prevention** is currently lookup-by-`lower(name)` only. The "did you mean X?" suggestion using pg_trgm similarity is `tasks/open/006`.

## Step 2 - Role

**Component:** `RoleStep` (`ContributeFlow.tsx:738-842`).

**Captures (required):**
- `role` (string, free text, trimmed).

**UX:**
- Typeahead -> `/api/roles/suggest?q=`, 250ms debounce.
- Suggestions come from a curated BD seed list first (`src/lib/common-roles.ts`); Gemini fallback is offline / cached.
- Server validation: non-empty after trim (`route.ts:98`).
- No length cap is enforced on the client; server simply trims (`:201`).
- Response carries `Cache-Control: public, s-maxage=60, stale-while-revalidate=300` (since 2026-05-19) so repeat keystrokes are served from the Vercel edge without invoking the function. `/api/companies?slug=` similarly carries `s-maxage=300` since slugs rarely change.

**On approval (later, in admin):** the free text is canonicalized into `roles` (creating if needed) and the alias added to `role_aliases`. See `docs/product/admin-moderation.md`.

## Step 3 - Time period

**Component:** `TimePeriodStep` (`ContributeFlow.tsx:850-968`).

**Captures (required):**
- `fromYear` (integer, 2000 - current year).
- `toYear` (integer, >= fromYear) **or** `null` for "Present" / "still working here".

**Client validation** (`:876-883`): selecting an invalid `toYear` after editing `fromYear` auto-clears it. The toYear dropdown filters to `>= fromYear`.

**Server validation** (`route.ts:108-111`):
- `role_start_year`: integer, >= 2000, <= current year.
- `role_end_year`: `null` (Present) **or** integer where `role_end_year >= role_start_year` and `>= 2000`.

**DB constraint** (migration `20260502150000`): `role_end_year is null or (role_end_year >= role_start_year and role_end_year >= 2000)`. Enforced at the schema level so a malformed payload can't slip past app code.

**UI:** live preview reads "2020 - Present" or "2020 - 2024".

## Step 4 - Salary

**Component:** `SalaryStep` (`ContributeFlow.tsx:970-1033`).

**Captures (required):**
- `salary` (integer, BDT, monthly, gross).

**Client UX:**
- Numeric input only (regex strips non-digits).
- Live thousand-separator formatting.
- "Don't include Eid bonus, WPPF, variable pay, or yearly bonus" microcopy on the helper line.
- Enter key advances.

**Server validation** (`route.ts:60-61`, `:112`, `:120-123`):
- Must be a positive integer.
- **Spam floor: >= 4,000 BDT.** Below that returns `400 { error: "salary_too_low" }`. The floor exists because <4,000 BDT/month is implausible for any tracked role and almost certainly fat-finger or test input.
- No upper cap. Aggregates downstream are robust to outliers (median, not mean).

**Client microcopy never promises moderation that depends on identifiers** (per user feedback memory). The framing is "we check before showing", not "we'll catch this if it's wrong".

## Step 5 - Pay day

**Component:** `PayDayStep` (`ContributeFlow.tsx:1035-1098`).

**Captures (required):**
- `payDay` (string). One of:
  - `"1"` ... `"31"` - day-of-month numeric.
  - `"first"` - first working day of the month.
  - `"last"` - last working day of the month.
  - `"varies"` - no fixed day.

**Stored as text on `salary_entries.disbursement_day`** so non-numeric values fit. UI is a 31-cell day grid plus three special buttons for "first / last / varies".

**Display formatting** (`src/lib/format.ts:77-128`, `formatPayDay`):
- Numeric values: ordinal in EN ("5th"), Western digit + locative particle in BN ("৫ তারিখ" or "৫ তারিখে").
- "first" -> "First working day" / "মাসের প্রথম কর্মদিবস".
- "last"  -> "Last working day"  / "মাসের শেষ কর্মদিবস".
- "varies" -> "Varies" / "নির্দিষ্ট নয়".
- Two output forms: **bare** (review card uses this: "5th") and **phrase** (company-page hero: "Paid on the 5th"). Bangla phrase form glues the locative `ে`.

## Step 6 - Delays

**Component:** `DelaysStep` (`ContributeFlow.tsx:1100-1138`).

**Captures (required):**
- `delayed` (boolean). "Was salary late in the last 12 months?"

Single Yes/No; auto-advances on selection. Stored as `salary_entries.delayed_last_year` (NOT NULL).

This single boolean is what drives the company-level delay aggregate; modal verdict (>= 50% yes) decides whether the company page shows "Often delayed" vs "No delays reported". See `docs/product/company-page.md`.

## Step 7 - Benefits (optional, all fields nullable)

**Component:** `BenefitsStep` (`ContributeFlow.tsx:1186-1379`).

**This is the step the previous version of this doc skipped.** Every field here is nullable; `null` means "didn't answer" and is excluded from aggregate denominators (so old pre-benefits entries don't drag the displayed totals down). Sub-fields require their parent boolean = true, enforced both in API validation (`route.ts:141-143`) and as DB CHECK constraints (`migration 20260505120000:25-31`).

The schema (`alter table salary_entries`):

```
performance_bonus       boolean
performance_bonus_freq  bonus_freq_t        -- enum: quarterly | annual | both
profit_sharing          boolean
profit_sharing_amount   int                 -- BDT, > 0 if present
health_insurance        boolean
health_insurance_cov    insurance_coverage_t
                        -- enum: self | self_spouse | self_spouse_kids
                        --       | self_spouse_kids_parents
provident_fund          boolean
gratuity_fund           boolean
```

### 7a. Performance bonus

- Yes / No pills.
- If Yes -> 3-option grid for frequency: **Quarterly**, **Annual**, **Both** (`bonus_freq_t` enum).
- If user toggles back to No, the frequency is auto-cleared (`:1213-1215`).
- Server check: `performance_bonus_freq` must be `null` OR `parent === true` AND value matches enum.
- Aggregated **per role** (not per company) on `roles.perf_bonus_yes / perf_bonus_total / perf_bonus_modal_freq` because perf bonus structure varies by role even within one org.

### 7b. Profit sharing

- Yes / No pills.
- If Yes -> text input for **last amount received in BDT** (live thousand-separator formatting).
- Server check (`route.ts:142`): amount may be `null` OR positive integer AND `parent === true`.
- DB CHECK (`migration 20260505120000:28-29`): `profit_sharing_amount is null or (profit_sharing = true and profit_sharing_amount > 0)`.
- Aggregated **per company** on `companies.profit_share_yes / profit_share_total / profit_share_median_amount`.

### 7c. Health insurance

- Yes / No pills.
- If Yes -> 4-option list for coverage:
  - **Self only** (`self`)
  - **Self + Spouse** (`self_spouse`)
  - **Self + Spouse + Kids** (`self_spouse_kids`)
  - **Self + Spouse + Kids + Parents** (`self_spouse_kids_parents`)
- Server check (`route.ts:143`): `health_insurance_cov` may be `null` OR `parent === true` AND match enum.
- Aggregated **per company** on `companies.health_ins_yes / health_ins_total / health_ins_modal_cov`.

### 7d. Provident fund

- Simple Yes / No, no sub-field. Aggregated on `companies.pf_yes / pf_total`.

### 7e. Gratuity fund

- Simple Yes / No, no sub-field. Aggregated on `companies.gratuity_yes / gratuity_total`.

**Skipping the entire step is allowed.** All fields default to `null`; the user can press Continue without picking anything. The only path that fails validation here is a malformed sub-field (frequency without bonus = true, etc.).

## Step 8 - Review + submit

**Component:** `ReviewStep` (`ContributeFlow.tsx:1427-1517`).

Read-only summary of every captured field with per-row Edit buttons. Benefits row uses `summarizeBenefits()` which produces a comma-separated summary or "Not provided" if all-null.

Submit triggers `POST /api/submit` with the assembled payload + the Cloudflare Turnstile token (invisible widget) + the `idempotency_key` (UUID, generated client-side and persisted in localStorage so it survives reloads, `:98-103`).

## Server-side: `POST /api/submit`

**File:** `src/app/api/submit/route.ts`.

**Pre-checks (in order):**

1. **Cookie soft rate limit** (ADR-0008): a cookie tracks per-browser submission count over a sliding window. HttpOnly, Secure (production), SameSite=Lax. Returns `429 { error: "rate_limited" }` over the cap. Counter increments only on success. Cap value, TTL, and cookie name intentionally redacted from the public mirror.
2. **Turnstile verification** (`src/lib/turnstile.ts`): if `TURNSTILE_SECRET_KEY` is unset -> short-circuit pass (test environments). Otherwise verify with Cloudflare; returns `403 { error: "turnstile_failed" }` on failure. When the E2E suite is rebuilt (`tasks/open/009-expand-e2e-coverage.md`), wire Cloudflare's published always-pass test keys so the browser never hits a real challenge.
3. **JSON shape + field validation** (`route.ts:97-143`):
   - Required: `role_freetext`, `role_start_year`, `monthly_salary_bdt`, `disbursement_day`, `idempotency_key` (opaque client-generated key), `delayed_last_year`.
   - Required `company_id` OR `company_name`.
   - Salary above an implausibly-low floor (`salary_too_low`).
   - Benefit sub-field constraints as described in step 7.
   - Failures return `400` with one of: `invalid_json`, `invalid_fields`, `missing_company`, `invalid_benefits`, `salary_too_low`.

**Insert + idempotency** (`route.ts:189-236`):

- INSERT with `ON CONFLICT (idempotency_key) DO NOTHING` against the unique partial index from `init_schema.sql:76-78` (`unique on salary_entries (idempotency_key) where idempotency_key is not null`).
- If the insert no-ops (duplicate idempotency key), look up the existing row and return its id with `status` from the row (do **not** re-trigger auto-approve on a retry).
- Default status = `'pending'`.

**Auto-approve** (live as of 2026-05-08, **not "planned"** as some older docs claim):

`tryAutoApprove(tx, entryId, ...)` in `src/lib/aggregate.ts:214-278`. Threshold constants intentionally redacted from the public mirror.

Conditions (all must hold):
1. The role exists (looked up by canonical name first, then alias).
2. The role has enough prior approved entries.
3. `role.salary_median is not null`.
4. The submission falls within the tolerance band of the median.

If approved: flip `status = 'approved'`, write to `auto_approve_log` (entry_id, role_id, role_median_bdt, submitted_bdt, band_pct, auto_approved_at), recompute aggregates, all in the same transaction. Admin can revert from `/admin/auto-approved` (`docs/product/admin-moderation.md`).

**Cache invalidation:**
- On auto-approval, the affected company slug pages are revalidated for both locales (`:267-269`).
- On normal pending insert, no public cache change.

**Aggregate recompute** runs **outside** the insert transaction (`:242-251`) to release the connection slot earlier - the recompute itself takes its own short txn.

**Response:**
- Success: `201 { id, status }` where status is `pending` or `approved` (auto-approved).
- Errors: `400` (input), `403` (turnstile), `429` (rate limit), `500` (`db_error`).

## Draft persistence

**localStorage key:** `bk_contribute_draft` (`ContributeFlow.tsx:12`).

**Draft shape:**

```ts
{
  company: { id: string | null, name: string } | null,
  role: string,
  fromYear: number | null,
  toYear: number | null,
  salary: number | null,
  payDay: string | null,
  delayed: boolean | null,
  benefits: Benefits,           // see step 7 schema
  idempotency_key: string,      // UUID; persisted across sessions for true idempotency
}
```

- **Saved** on every change after hydration if the draft is "partial" (any field touched, `:233-234`).
- **Cleared** on submit success (`:333`) or via the "Start fresh" button on the resume banner.
- **Backward-compat:** drafts written before benefits shipped auto-hydrate with empty benefits (`:115`).

The **idempotency key persists across sessions**, so a user who reloads mid-submit and then completes the flow doesn't double-submit on a network retry.

## Edge cases

- **localStorage cleared mid-flow:** silent restart from card 1; we don't try to recover.
- **Cookie soft cap hit:** server returns 429 with friendly copy; user retries tomorrow or in private browsing.
- **Turnstile fails:** failure card with a "try again" CTA; no auto-retry.
- **Idempotent retry after success:** server returns 200/201 with the existing entry id; no duplicate inserted.
- **Company name resolution race:** two simultaneous "+ Add new" submissions for the same name are reconciled by `ON CONFLICT (slug) DO UPDATE`.
- **Auto-approve narrowly misses (e.g., n=2 instead of 3):** entry stays `pending`; admin reviews normally.
- **Edit from review and back:** later cards re-validate when their inputs change; the user can't ship a stale combination.
- **Step regression:** tapping a previous card always allows editing.

## Privacy framing

Microcopy on each step references the trust pillars (anonymous, not stored, hardgate). Per user auto-memory: **never imply moderation mechanisms that would require identifiers we don't store** (no "we'll catch you if you submit something fake" - we won't, by design).

## Out of scope (v1)

- Edit / delete own submission via cookie token (`tasks/open/030`).
- Right-of-reply prompt for the company (`tasks/open/029`).
- Bangla numeric input (ADR-0007; toggle deferred in `tasks/open/027`).
- "Early pay" semantics (current-month-pays-next-month) - schema in `tasks/open/019`, UX in `020`.
- Role typeahead matching on Bangla aliases (`tasks/open/028`).
- Server-side normalization of company / role text (`tasks/open/018`).

## Related

- ADRs: 0001 (no em dash), 0003 (no PII), 0004 (hardgate framing), 0007 (canonical English data), 0008 (cookie soft rate limit)
- Files: `src/app/[locale]/contribute/`, `src/app/api/submit/route.ts`, `src/app/api/roles/suggest/route.ts`, `src/lib/turnstile.ts`, `src/lib/aggregate.ts` (`tryAutoApprove`), `src/lib/format.ts` (`formatPayDay`)
- Migrations: `20260501183033_init_schema.sql`, `20260502150000_replace_years_experience_with_role_years.sql`, `20260503000000_add_entry_source.sql`, `20260505120000_add_benefits.sql`
- Mockups: `design/All Screens/bk-mockups.jsx` (MobileContribute), `bk-flows.jsx` (full storyboard)
- Tickets: 006 (duplicate prevention), 011 (likes/dislikes), 018 (sanitize text), 019/020 (early pay), 027 (Bangla numerals), 028 (Bangla role aliases), 030 (edit own)
