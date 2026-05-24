# Beton Kemon — Redesign Prompt (copy-paste into your AI design tool)

Everything below this line is the prompt. No attachments needed — every fact about the product, audience, current screens, and constraints is in-text. Copy from the next line down.

---

You are the lead designer at a world-class studio (think Pentagram crossed with Stripe's brand team and Apple's Human Interface group). You have just been handed the redesign of a live consumer product. The current product works but looks like a tasteful Tailwind starter — clean, but not a brand. Your job is to give it a point of view.

Read this entire brief before producing anything. Then deliver in the exact format specified at the end. Do not ask clarifying questions; where the brief is silent, make a confident defensible choice and explain it in one line.

<product>
**Name:** Beton Kemon (Bangla: বেতন কেমন, lit. "How is the salary?")
**Live URL:** https://www.betonkemon.com
**One-liner:** The first crowdsourced, anonymous salary-transparency site for Bangladesh. People share what they earn, what day they get paid, whether their salary gets delayed, and what benefits they get. We aggregate this into per-company pages.
**Why it exists:** A young professional in Dhaka cannot answer "is this offer fair, and will I actually get paid on time?" Glassdoor BD is empty, Levels.fyi is FAANG-only, LinkedIn salary is global noise. We are the local answer.
**What we are not:** We are not Glassdoor (reviews). We are not Levels.fyi (FAANG comp). We are signals: salary range, pay-day reliability, delay rate, benefit truth.
**Tagline candidate (you may replace):** "Anonymous salary transparency for Bangladesh."
**Brand-name etymology:** *Beton* = salary, but in Bangla it also literally means *concrete*. There is a brutalist-soft register here — solid, weight-bearing, warm — that no global salary site has claimed. Worth exploring as visual material without becoming literal.
</product>

<audience>
- **80% mobile**, mid-range Android in Dhaka and Chattogram on patchy 4G. Optimize mobile first; desktop is a polish layer.
- **Demographics:** young professionals 22–35, early-career to mid-career, often first-jobbers or first-job-switchers. High Bangla literacy, moderate-to-good English.
- **Visual fluency:** they use Pathao, bKash, Daraz, Foodpanda, Chaldal daily. Generic-looking sites read as "another sketchy career portal" and bounce.
- **Trust posture:** they are about to type their salary into the site. If it feels cheap, NGO-flavored, or foreign, they leave. If it feels like a serious institution without being corporate-cold, they submit. We currently convert ~16% of visitors into contributors — must not regress.
- **Reading direction:** LTR. Bangla and Latin coexist on the same screen. Bangla is taller and rounder than Latin; vertical rhythm must absorb both without one feeling like an afterthought.
</audience>

<current_state_described>
Since you cannot see the live site, here is a precise textual description of every screen as it ships today. Use this to know what we have and what we are willing to throw away.

**Landing (mobile, /en):**
A header bar with a small terracotta-square wordmark on the left ("Beton Kemon") and an EN | বাংলা language toggle on the right. Below: a search input with magnifying-glass icon and placeholder "Search companies...". Then two stat tiles side-by-side showing "1,457 companies" and "6,568 salaries shared" (the second number in terracotta). Below that: a row labeled INDUSTRY with horizontally-scrolling chips (Aviation, Banking, Conglomerate, Consulting, ...). Below that: a row labeled SALARY with four mutually-exclusive band chips (Under 50k, 50k–100k, 100k–200k, 200k+). Then a section heading "Companies" with a "Sort: Most shared / Show: 10" control. Then a vertical list of company cards. Each card: small square logo on the left, company name top-right, a metadata line "126 entries · 29 roles · 8k–379k BDT". A floating round terracotta button bottom-right says "+ My Salary". Pagination at the bottom (1, 2, ..., 146). Footer: "Powered by: Grow Your Career", a one-line disclaimer, and About / Contact / Privacy links.

**Landing (desktop):**
Same single-column layout as mobile, just centered in ~600px in the middle of a beige page. The ~840px of rails on either side are wasted. This is one of the bigger misses.

**Company page (mobile, e.g. /en/c/bkash):**
Hero: small back chevron + "Back" on the left, EN | বাংলা on the right. Below: company logo (red bKash square), company name "bKash Limited", a "Fintech" pill. A two-stat strip: "126 entries · 8k–379k" (the "8k–379k" prominent). Below: a section labeled COMPANY with a 3×2 grid of benefit cells: Pay day "20th", Delays "no delays", Health insurance "Self + Spouse + Parents", Profit sharing "৳241,925", Provident fund "Yes", Gratuity "Yes". Each cell has a small icon, a label, and a value. Below: a section labeled ROLES with a long vertical list of role cards. Each role card: role title (e.g. "bKash Intern"), salary range "8k–12k BDT", subtitle "Based on 1 person", a footnote "Add my salary for this role →" linked, and "0 of 1 get a performance bonus" if data exists. The list goes 30+ deep on bKash. Floating "+ My Salary" FAB bottom-right. The terracotta accent shows up only in the FAB and the few links.

**Company page (desktop):**
The only screen with a real two-pane: a left sidebar lists the top 20 popular companies (avatar + name + "N roles · M entries"), the current company highlighted with a left border. The main pane is the same content as mobile, slightly wider. Sidebar exists nowhere else in the product.

**Onboarding (mobile, 3 cards):**
Card 1 of 3 — "আপনার ভাষা?" / "Pick your language". Two large radio cards: "বাংলা — Bangla · default" and "English — Latin script". A primary terracotta "Continue →" button at the bottom and a small "You can change this later" line.
Card 2 of 3 — "Why this exists" — three numbered bullet points explaining the salary opacity problem.
Card 3 of 3 — "How it works" — three numbered trust pillars: (1) "No signup. Ever." (2) "Pay day and delay trends unlock at 5+ entries." (3) "We check before showing." Final CTA: "Get started".

**Contribute (mobile, 8 sequential cards):**
Top bar: back chevron, "1 OF 8" centered, close X right. A thin terracotta progress bar under the top bar. Then: a question heading ("Which company?"), a one-line subtitle ("Type in English. We'll show matches as you go."), one input or one selection control. Each card asks ONE thing. The 8 steps: (1) Company picker, (2) Role typeahead, (3) Time period (from-year + to-year/Present), (4) Monthly salary (BDT, integer, ≥4,000), (5) Pay day (a 31-day grid + "first / last / varies"), (6) Was salary late in last 12 months? (Yes/No), (7) Benefits (5 optional sub-questions), (8) Review + submit. Drafts auto-save to localStorage. There is no signup, no email, no phone — ever.

**Submit success:**
Currently a generic "Thanks, your submission is being reviewed" card. Underwhelming. This is the only point at which the user gives the platform something — should feel like a quiet thank-you, not confetti.

**What the current design gets right (preserve):**
- Calm, low-chroma surface. Money is sensitive; no casino energy.
- Mobile thumb targets and Bangla line height are respected.
- Honest empty states. We never fake data to look fuller. The "5 entries needed to unlock pay-day signal" is part of the brand.

**What the current design gets wrong (must improve):**
- No memorable mark, no signature interaction, nothing recognizable at 100ms. It could be any Tailwind starter.
- Desktop is a wasted ~1100px. Two-pane only on company page.
- The hero buries the magic. The two stat tiles ("1,457 companies, 6,568 salaries shared") are the most credibility-bearing things on the page; they look like small numeric tiles, not a moment.
- The benefit grid reads like a spec sheet. The data deserves to land like a *verdict* — "this place pays on the 5th, no delays" is genuinely valuable and should feel revealing.
- Filters are flat chips with no information design.
- Typography is one weight stack doing five jobs. Numerals do not feel special even though numerals are the product.
</current_state_described>

<user_journeys>
The five journeys, ranked by traffic.

**A. Curious arrival → company lookup (highest volume).** URL → onboarding once → landing → search a specific company → company page → leave. Target: under 30 seconds from URL to "I have my answer." Search is the hottest endpoint by 4x.

**B. Curious arrival → browse by category (growth path).** Same arrival, no target company. They want "what do banks pay?" or "what does Marketing make at FMCGs?" Today they can filter by industry and salary band only.

**C. Convinced visitor → contributor.** 8-step contribute flow on mobile. Single-purpose cards with autosaved draft. Submit-success is the brand's emotional climax.

**D. Returning visitor.** Comes back two weeks later to recheck a company. Wants to feel the page has *life* (recent submissions, freshness) without us showing PII. Room for a subtle "as of" line, sparkline, or "3 new entries this week" pill.

**E. Anchor employer / journalist / HR exec.** Lands on a company page from a press article or LinkedIn share. The page must look broadcast-quality. Design assuming it ends up in a Daily Star article, a LinkedIn carousel, and an HR offboarding deck — same week.
</user_journeys>

<near_term_features>
Design for these now, do not punt them to v2:

1. **Search by department** — Marketing, Finance, Engineering, Sales, Operations, HR. Mapping data exists; UI does not. Results need to surface aggregate cards: "Department: Marketing — 47 companies, salary range 25k–280k BDT".
2. **Search by role title** — "Territory Officer", "Relationship Executive", "Senior Software Engineer". Cross-company comparison: "Territory Officer at 23 companies, median 45k BDT, range 28k–95k". This is genuinely new — no Bangladeshi competitor surfaces this.
3. **Search becomes tri-modal** — companies / departments / roles. Today it is company-only. Pick a paradigm — chips, tabs, a unified ranked dropdown with type badges, your call.
4. **Department label on role cards.**
5. **Right-of-reply from companies** (v2 polish; design role-card region with room for it).
6. **Trending signals** — recent submission velocity, newly-unlocked companies. We have the timestamps; we just don't surface them.
</near_term_features>

<brand_soul>
Be in the same conversation as — do not copy:
- **Apple iOS Wallet** — the way a credit card sits on the screen. Calm, generous, single confident hierarchy.
- **Monzo / Revolut** — finance products that feel human. Numbers as a typographic moment.
- **NYT Election Needle / Upshot graphics** — data presented with quiet drama. The shape of the data *is* the design.
- **Linear** — restraint. Surface gradients used as architecture, not decoration.
- **Mailchimp (Pentagram)** — a small fun mark inseparable from the product, without being twee.
- **MoMA wayfinding** — institutional weight. Users should feel the trust they would feel reading a Bangladesh Bank statement, not a startup deck.

Words a stranger should feel within 5 seconds:
calm, modern, honestly Bangladeshi, trustworthy, confident with numbers, gives a damn about typography, made by someone who lives here.

Words they should never feel:
sketchy, NGO-template, generic SaaS, foreign tech-bro, gambling-app, government-portal, AI-generated dashboard.

Bangladeshi cues you may lean into (subtly):
- The materiality of concrete (the brand name). A brutalist-soft register, solid and warm.
- The visual language of receipts and pay slips in BD — the carbon-copy duplicate, the rubber-stamp, the taka glyph (৳).
- Dhaka's late-afternoon light, monsoon palette, the warm beige of old-Dhaka brick.

Cues to avoid:
- Saffron/orange tropical "South Asia" treatment.
- Rickshaw art / nakshi kantha textile patterns as decoration.
- Cricket.
</brand_soul>

<hard_constraints>
These are non-negotiable. Design around them, not against them.

1. **No emoji anywhere in production UI.** Build out an icon set instead. Stroke-based, single-weight.
2. **No em dash (—) in user-facing copy.** Plain hyphen only. This affects every microcopy string you write.
3. **Bangla-default, English opt-in.** /bn/... is canonical, /en/... is opt-in. Bangla and Latin must be designed as first-class siblings — Bangla is not a translation skin.
4. **Bangla typeface:** Hind Siliguri today. You may propose a replacement, but do not propose Noto by default — too generic. **Latin typeface:** Geist today, you may repick. **Numerals stay Western digits in both locales** (Bangla numerals are out of scope for v1).
5. **No purple, no violet** as brand color. Beyond that the palette is yours.
6. **Mobile-first.** Design mobile, then earn every desktop-only element. Do not design desktop and shrink it.
7. **Anonymous by design.** No avatars, no usernames, no likes-on-an-entry, no "verified contributor" badges. We have no user accounts and we will not get them. The design must feel *populated* without ever implying individual identity. This is one of the hardest creative problems in the brief.
8. **The ≥5 hardgate is a brand pillar, not a bug.** When a company has fewer than 5 entries, pay-day / delay / benefit signals are *locked* with a "need 4 more" empty state. Locked states must feel honorable — like an honest "we do not know yet" — not punitive. **Avoid blur-and-paywall metaphors;** that connotes paid content and we are free.
9. **Trust copy must not promise moderation that depends on identifiers we do not store.** Never say "we'll catch fakes" or "we verify who you are." Say "we check before showing."
10. **Per-company brand colors exist.** Each company can have a hex set in admin, used today as the avatar background and a small accent strip on its hero. Design a system that gracefully absorbs an arbitrary brand color (bKash pink, Brac red, Unilever blue) without the rest of the page fighting it. Auto-flip text color to maintain WCAG contrast.
</hard_constraints>

<principles>
Use these to break ties.
1. **The number is the hero.** If the salary numbers are not the loudest thing on the screen, the design is wrong.
2. **Calm under uncertainty.** A page that says "we don't know yet, we need 4 more entries" should feel as polished as one with 800 entries.
3. **Anonymous but populated.** No individual identity, but the crowd must be present. Use counts, ranges, recency, distribution.
4. **Bangla and Latin are first-class siblings.**
5. **Mobile is the design; desktop is the encore.**
6. **Restraint over richness.** Two great moments per screen beat seven mediocre ones.
7. **Honest density.** This product has a lot of data. Hiding it behind tabs and accordions is cowardly.
8. **Distinct, not derivative.** If the design could ship as Glassdoor BD, Levels.fyi BD, or Payscale BD with a logo swap, it has failed.
</principles>

<creative_freedom>
We are deliberately NOT specifying — make confident calls:
- The full color palette (only constraint: no purple/violet).
- Typeface choices (with the noted current state).
- Whether to keep pill-and-grid filters or replace them.
- Whether the company hero is a band, card, full-bleed wash, or something none of these.
- Whether salary ranges are text, histograms, beeswarms, or a custom mark.
- Whether the "popular companies" sidebar survives.
- Whether the contribute flow stays 8 single-purpose cards or restructures.
- The exact form of the signature element.

If you make a confident, defensible choice, we will follow you. If you ask, we will pick the safest option and the result will be average.
</creative_freedom>

<deliverables>
Two outputs. Both required.

## Output A — Comprehensive design system

A self-contained, documented system. Cover all of:

- **Foundations**
  - Color tokens with semantic names: surface, surface-elevated, border, border-strong, text-primary/secondary/tertiary, accent, accent-hover, success, warning, danger, info, and a *locked / awaiting-data* token for the ≥5 hardgate state. Both **light and dark mode** at the token level. (Dark UI ships in v2 but the system must be ready.)
  - Typography ramps for **both scripts side-by-side**, not just Latin. Specify size / line-height / weight at every step. Optionally a third face used only for salary numerals — tabular, confident; the typographic statement of the brand.
  - Spacing scale.
  - Border radius scale.
  - Elevation / shadow language.
  - Motion principles — exact durations, easings, and *meaning* of each motion. Specify `prefers-reduced-motion` fallbacks.
  - Iconography — stroke-based, single-weight. At minimum: search, company, department, role, salary, calendar (pay-day), clock (delays), shield (insurance), coin (profit-share), bank (PF), gift (gratuity), trending, locked, language toggle, close, back, edit, check. Around 20 icons.
- **Components** — every state (default, hover, focus, active, disabled, loading, error, empty, locked):
  - Button — primary, secondary, ghost, destructive
  - Text input, numeric input, search input
  - Pill / chip (selectable + filter variants)
  - Cards — company card, role card, benefit cell, search result row
  - Accordion (filter pattern)
  - Dropdown / select
  - Modal + mobile bottom-sheet
  - Toast / inline notification
  - Progress bar (contribute flow)
  - Pagination
  - Locked / awaiting-data placeholder
  - Salary-range visualizer (the brand-defining viz primitive)
- **Data-viz primitives** — salary distribution (histogram / beeswarm / your call), pay-day calendar mark, delay-rate indicator, cross-company comparison row.
- **Signature element.** Pick one thing — a chart style, a number treatment, a transition, a card chrome, the ৳ glyph, the way locked cells look — and make it unmistakable.
- **The mark.** Wordmark + app-icon-scale glyph. Currently the Bangla letter ব in a colored square. Keep, refine, or replace. Must read at 16px (favicon) and 1024px (App Store) without being two different identities.
- **OG / share card** template. 1200×630 per company, with a per-company brand-color absorption rule.
- **Documentation surface.** A design-system page (or pages) showing tokens, components in all states, typography ramps in both scripts, motion examples, do/don't pairs.

## Output B — Every user journey, mobile + desktop

For **every screen below**, deliver **mobile (390×844)** and **desktop (≥1440 wide)**. Mobile is canonical. Desktop is not a centered phone column — it must be a real, considered design.

Include all critical states (empty, loading, error, locked, success).

**Onboarding (3 cards + transition)**
- Card 1 — Language pick (Bangla default, English opt-in)
- Card 2 — Purpose
- Card 3 — Trust (3 trust pillars: anonymous, hardgate, "we check before showing")
- The exit transition into the landing page

**Search (the heart of the redesign)**
- Empty focus state with recent searches
- Typing — company results
- Typing — department results (new axis, e.g. "Marketing")
- Typing — role results (new axis, e.g. "Territory Officer")
- Mixed-results state (companies + departments + roles in one query)
- Zero results / "search miss" with "+ add first salary for X" CTA
- Rate-limit / "search is busy" state
- **Department landing page** (e.g. "Marketing in Bangladesh") — cross-company aggregate, salary distribution, top employers, role breakdown
- **Role landing page** (e.g. "Territory Officer in Bangladesh") — salary distribution across companies, employer list, range and median
- **Browse / landing screen as the search hub** — hero, tri-modal search, aggregate stats moment, filter system that absorbs the new department + role axes, sorted result list, all empty states (day-0, filter zero-match, first-time visit before onboarding)

**Contribute (8 steps + auxiliaries)**
- Step 1 — Company picker (with "+ add new company" path)
- Step 2 — Role typeahead
- Step 3 — Time period (from-year + to-year / Present)
- Step 4 — Salary input (BDT, monthly, integer, ≥4,000 BDT)
- Step 5 — Pay day (31-cell grid + first / last / varies)
- Step 6 — Delays (Yes / No)
- Step 7 — Benefits (5 sub-fields: performance bonus + freq, profit-sharing + amount, health insurance + coverage tier, provident fund, gratuity — all optional, conditional sub-pickers)
- Step 8 — Review + submit
- Resume-draft banner state
- Submit-success moment (the brand's emotional climax)
- Rate-limit / turnstile-fail / network-fail sad states

**Company page**
- Hero variant 1 — brand color + logo
- Hero variant 2 — brand color + initial avatar (no logo)
- Hero variant 3 — neutral (no logo, no brand color)
- Benefit grid (or your replacement) — fully unlocked
- Benefit cells partial unlock — some cells unlocked, some "N more needed"
- Day-0 hardgate — company exists, fewer than 5 entries — the *honorable* "we don't know yet" treatment
- Role cards — one with a performance-bonus row, one without
- Sparse company — 6 roles
- Dense company — 80+ roles (real example: bKash Limited, ranges from 8k–12k BDT for interns to 359k–379k BDT for general managers)
- Desktop two-pane — sidebar of popular companies, or your replacement for the sidebar paradigm
</deliverables>

<format_expectations>
- High-fidelity mockups, not wireframes.
- All public-facing copy in **both Bangla and English**. Bangla is not a follow-up — design Bangla strings as first-class. (You may propose new microcopy in either language.)
- Annotate non-obvious interactions, motion, and the rationale behind your signature design decisions. One short note per non-obvious choice.
- Specify dimensions and spacing on every screen.
- Specify exact tokens used (do not bake in literal hexes in component specs).
</format_expectations>

<evaluation_criteria>
We will judge the result, in order:
1. **Does the company page feel like a verdict?** A glance should yield "I know what working there is like, financially." If it reads as raw data, it has failed.
2. **Does the search feel inevitable?** When the new department + role axes land, the interaction should feel like it was always meant to be there.
3. **Does the contribute flow feel like a transaction with a trustworthy institution?** Calm progress, honest copy, a quiet thank-you at the end.
4. **Would a Dhaka-based designer screenshot it?** Local pride is a metric. They should feel "finally, a BD product that doesn't look outsourced."
5. **Would Apple ship it?** The proof of restraint. If you cannot picture the screens in a Today-tab editorial card on the App Store, the design has not earned its weight.
</evaluation_criteria>

Begin by stating, in 3-5 sentences, the *point of view* you have chosen for the brand (the design thesis — what makes Beton Kemon visually itself). Then deliver Output A (the design system), then deliver Output B (the journeys, in the order: Onboarding → Search → Contribute → Company page), each with mobile and desktop side-by-side. Conclude with a one-page rationale for your three biggest decisions.

Do not ask clarifying questions. Make confident defensible choices. Begin.
