# Design System

Beton Kemon's visual language. Two parallel sources of truth:

- **`src/app/globals.css`** is the authoritative token definition that ships in production. If a number, color, or font appears in a component, it must trace back to a `--bk-*` token here.
- **`design/`** is the reference workshop. JSX mockups, an interactive design-system HTML, and a PDF print export. Not part of the build (eslint-ignored, runs via Babel-standalone in a browser); used for visual truth, design exploration, and onboarding new contributors.

Both must agree. If `design/` and `globals.css` drift, fix `globals.css` and update the affected components - the production tokens are what users actually see.

## Tokens (cite from `globals.css`, not memory)

The full set lives in `src/app/globals.css` under `:root`. As of this writing:

**Surface / chrome**
- `--bk-bg` (#faf8f5) - page background
- `--bk-bg-elev` (#ffffff) - cards, modals, dropdowns
- `--bk-border` (#e8e4dd) - default border
- `--bk-border-strong` (#d8d2c8) - emphasized border (accordion dividers, focus)

**Text**
- `--bk-text-pri` (#1a1a1a) - body, headings
- `--bk-text-sec` (#6b6b6b) - meta, helper copy
- `--bk-text-ter` (#9a9a9a) - timestamps, deprecated labels

**Accent + state**
- `--bk-accent` (#c8553d - terracotta) - primary CTA, brand highlight
- `--bk-accent-hover` (#a6432f) - CTA hover
- `--bk-warning` (#d4a017) - delay indicators, hardgate locked state
- `--bk-warning-soft` (rgba) - warning background tint
- `--bk-success` (#4a7c59) - "no delays" indicator, approved states

**Per-department tint palette** (`--bk-cat-{dept}-bg` + `--bk-cat-{dept}-fg`, one pair per `Department` enum value): each department surface (`DeptIcon` container on the departments list, role-row icons, drilldown headers) reads its tint via `getDeptTint(dept)` in `src/lib/dept-tint.ts`. Hues are differentiated across 19 muted tones (teal, indigo, rose, cobalt, coral, gold, emerald, sky, slate, forest, terracotta, navy, olive, rust, steel, burgundy, mustard, taupe, espresso). Bg is ~10% opacity, fg targets AA on the resulting tint at 20px. Hard rule 8 (no purple/violet) is honored — rose/pink replaces what would otherwise be a violet slot. Do not hand-pick a category color at a callsite; route through `getDeptTint`.

**Typography variables** (set via `next/font` in `[locale]/layout.tsx`, not literal):
- `--bk-font-bn` - Hind Siliguri (Bangla)
- `--bk-font-latin` - Geist (Latin)

**Shadow** (one elevation, used everywhere a card lifts):
- `--bk-shadow: 0 1px 2px rgba(0,0,0,0.04), 0 4px 12px rgba(0,0,0,0.04)`

Tailwind v4 maps these into `@theme inline` so `bg-bk-bg`, `text-bk-text-pri`, `border-bk-border` utilities work directly.

## The hard rules in token form

(See `docs/conventions.md` for the rationale; this section is the operational form.)

- No purple, no violet. Brand vocabulary is greens, neutrals, terracotta accent.
- No `system-ui` as primary - Geist (Latin) and Hind Siliguri (Bangla) only.
- One shadow elevation, total (`--bk-shadow`).
- Spacing on the 4px scale: 4 / 8 / 12 / 16 / 24 / 32 / 48 / 64. Nothing else.
- Border radius: 4 (inputs/small buttons) / 8 (cards/large buttons) / 16 (modals only).
- Three motion primitives: 80ms card slide, 150ms search-result fade, 200ms submit-success scale. Honor `prefers-reduced-motion`.

## `design/` folder map

```
design/
  Beton Kemon - Design System.html         Self-contained interactive design system
  All Screens/
    index.html                             Loader for the JSX mockups (Babel-standalone)
    Beton Kemon - Design Exploration       The canonical visual reference for visual QA
      (Print).pdf                          Open this when reviewing whether a screen
                                           "looks right".
    bk-mockups.jsx                         Phone-frame mockups of every public screen:
                                           landing, company page, contribute steps, etc.
    bk-flows.jsx                           Multi-screen journey storyboards (onboarding,
                                           contribute end-to-end, search-to-company).
    bk-app.jsx                             Composes the canvas: accent / warmth / typeface
                                           tweaks bound to live --bk-* variables.
    design-canvas.jsx                      Layout primitive - the canvas that renders
                                           multi-phone storyboards.
    tweaks-panel.jsx                       Live-tweak sidebar for accent / warmth / type.
    ios-frame.jsx                          iPhone bezel / status bar / home indicator.
```

**How to run locally:**

```bash
# Open in any browser; everything is client-side via Babel-standalone:
open 'design/Beton Kemon - Design System.html'
open 'design/All Screens/index.html'
```

No build, no install. The mockups read `--bk-*` tokens from inline `<style>` and let you mutate them via the tweaks panel - useful for evaluating accent shifts before touching `globals.css`.

## When to use which artifact

| You want to... | Open |
| --- | --- |
| Check whether a built screen matches the intended visual | `design/All Screens/Beton Kemon - Design Exploration (Print).pdf` |
| See an interactive token / component reference | `design/Beton Kemon - Design System.html` |
| Read the source-of-truth CSS variables | `src/app/globals.css` (the `:root` block) |
| Storyboard a new flow before coding | Edit `design/All Screens/bk-flows.jsx` and reload |
| Evaluate an accent / warmth tweak across screens | `design/All Screens/index.html` + tweaks panel |
| Add or remove a token | `src/app/globals.css` first; then update `design/` to match |

## The 12-icon SVG set

Production icons live in `src/components/Icon.tsx`. **No emoji anywhere in production UI.** If a needed icon isn't in the set, add it to that file rather than reaching for an emoji or a one-off inline SVG.

Lucide-style stroke. Match the existing weight + cap when extending.

## Coupling rules

- **Components use tokens, not literals.** A button doesn't `bg-[#c8553d]` - it `bg-bk-accent`.
- **`design/` is reference, not import.** Production code never imports from `design/`. The eslint config explicitly ignores `design/**` because it runs in-browser via Babel-standalone with shared globals.
- **Token additions need a sync pass:** add to `globals.css`, regenerate any affected `design/` JSX, eyeball the print PDF for regressions.

## Related

- `docs/conventions.md` - copy + code conventions (the rationale layer)
- `docs/product/` - user journeys; each one references the screen mockup it implements
- `src/components/Icon.tsx` - the only icon vocabulary
- `src/app/globals.css` - the authoritative token block
