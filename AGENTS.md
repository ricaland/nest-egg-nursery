# Nest Egg Nursery — Agent Context

This file gives Claude the context needed to edit `index.html` in this repo.

## What this project is

A single-file web app (`index.html`) that projects investment account growth for 23 grandchildren over 40 years. It is hosted on GitHub Pages at `https://ricaland.github.io/nest-egg-nursery` and is entirely self-contained — no build step, no framework, no backend, no dependencies except Google Fonts and Chart.js loaded from CDN.

The intended audience is the family: the grandparent (GP) who is funding the accounts, and eventually each grandchild (GC) who can see their own projection.

---

## File structure

Everything lives in `index.html`. It is organized in three sections:

1. **`<style>` block** — all CSS using custom properties (CSS variables) defined in `:root`
2. **`<body>` / HTML** — the header, sticky nav, and six tab panels (each a `<div id="panel-*">`)
3. **`<script>` block** — all JavaScript at the bottom; no external JS files

There are no other source files. Do not split into multiple files unless explicitly asked.

---

## Financial model — how it works

### Grandchild accounts (23 total)

| Parameter | Value | Where set |
|-----------|-------|-----------|
| Number of accounts | 23 | `const N=23` |
| GC 1 start date | June 2026 | `const GC_STARTS` array |
| Each subsequent GC | +1 month later | GC 23 starts April 2028 |
| GP initial deposit | $1,000 (one-time, month 1 only) | `sl-gpinit` slider, default 1000 |
| GC monthly contribution | $20/month, starts month 1 | `sl-gcmonth` slider, default 20 |
| Annual rate of return | 7% (adjustable) | `sl-rate` slider |
| Projection length | 40 years (adjustable 10–50) | `sl-years` slider |

**Match events** — funded out of the GP Investment Account:

| Event | Default trigger | Match formula | Cap |
|-------|----------------|---------------|-----|
| Match 1 | Month 30 (2.5 yrs) | 25% of account balance | $500 |
| Match 2 | Month 60 (5 yrs) | 25% of account balance | $1,000 |

Grandchild qualifies for both matches automatically because they contribute $20/month (GC monthly contribution > 0 satisfies the eligibility condition).

**Compounding:** Monthly, using `(1 + annualRate)^(1/12) - 1` as the monthly rate. Applied to `(beginningBalance + totalDeposit)` each month.

### GP Investment Account

- Funded with **$1,000/month starting May 2028** (the month after GC 23 is initially funded in April 2028)
- Earns the same rate of return as GC accounts
- **Match payments flow OUT of this account** into each GC account at their respective match months
- The account **can go negative** — a negative balance signals the need for additional external funding
- Growth is only applied to positive balances: `Math.max(begBal + net, 0) * mRate`
- Runs for 40 years (480 months) from May 2028

---

## JavaScript architecture

### Key constants
```js
const N = 23;                          // number of grandchildren
const GC_STARTS = [...]                // array of {y, m} for each GC's start date
const GP_START = { y: 2028, m: 5 };   // GP Investment Account start
```

### Core functions

| Function | What it does |
|----------|-------------|
| `getP()` | Reads all slider values, returns a params object `p` |
| `projectGC(p, i)` | Projects one grandchild account; returns rows + summary stats |
| `projectGPAcct(p, gcRes)` | Projects the GP Investment Account; pulls match amounts from `gcRes` |
| `buildCF1(p, gcRes)` | Builds Section 1 of the cash flow tab (only months with activity) |
| `update()` | Master function — called on every slider change; runs all projections and re-renders every visible panel |
| `renderSummary(p, gcRes)` | Populates the Summary tab |
| `renderChildButtons(p, gcRes)` | Renders the 23 child selector buttons |
| `renderChildDetail(p, gcRes)` | Populates the Individual Accounts tab for `selChild` |
| `renderGPAcct(p, rows)` | Populates the GP Investment Account tab |
| `renderCF(p, cf1Rows, gpRows)` | Populates both sections of the GP Contributions tab |
| `renderChart()` | Renders the Chart.js line chart |
| `switchTab(tab, btn)` | Shows/hides panels, triggers chart render if needed |

### Global state (on `window`)
```js
window._p       // current params object
window._gcRes   // array of 23 projection result objects
window._gpRows  // GP Investment Account row array
let selChild    // index (0–22) of currently selected grandchild
let gChart      // Chart.js instance (destroyed and recreated on each render)
```

### Params object `p` shape
```js
{
  gpInit,    // GP initial deposit per GC account ($)
  gpMonth,   // GP Investment Account monthly deposit ($)
  rate,      // annual rate of return (decimal, e.g. 0.07)
  years,     // projection length in years
  gcMonth,   // GC monthly contribution ($)
  m1mo,      // match 1 trigger month (integer)
  m1pct,     // match 1 percentage (decimal, e.g. 0.25)
  m1cap,     // match 1 cap ($)
  m2mo,      // match 2 trigger month (integer)
  m2pct,     // match 2 percentage (decimal)
  m2cap,     // match 2 cap ($)
}
```

### GC result object shape (returned by `projectGC`)
```js
{
  rows,    // array of monthly row objects (see below)
  m1,      // match 1 amount received ($)
  m2,      // match 2 amount received ($)
  cumGP,   // cumulative GP contributions including matches ($)
  cumGC,   // cumulative GC contributions ($)
  bal5,    // balance at year 5 (row index 59)
  bal10,   // balance at year 10 (row index 119)
  bal20,   // balance at year 20 (row index 239)
  bal40,   // balance at year 40 (row index 479)
}
```

### Monthly row object shape
```js
{
  mo,        // month number (1-based)
  y, m,      // calendar year and month
  gpInit,    // GP initial deposit (only > 0 in month 1)
  gcDep,     // GC deposit this month
  match,     // match received this month (0 except at m1mo and m2mo)
  totalDep,  // gpInit + gcDep + match
  begBal,    // balance at start of month
  growth,    // investment gain this month
  endBal,    // balance at end of month
}
```

---

## HTML panels

| Panel ID | Nav tab label | Contents |
|----------|--------------|----------|
| `panel-summary` | Summary | Metric grid, avg-contribution banner, full 23-row table |
| `panel-assumptions` | Assumptions | All sliders (GP contributions, GC contributions, match events) |
| `panel-child` | Individual accounts | Child selector buttons + detail card + month-by-month table |
| `panel-gpacct` | GP investment account | Metric grid + full monthly detail table |
| `panel-cashflow` | GP contributions | Section 1 (initial deposits) + Section 2 (GP inv. acct flow) |
| `panel-chart` | Growth chart | Mode toggles + Chart.js canvas |

Only one panel is visible at a time (`display:block` via `.active` class). Tab switching is handled by `switchTab(tab, btn)`.

---

## CSS design system

All colors are CSS custom properties on `:root`. Do not hardcode hex values in new code — use variables.

| Variable | Value | Used for |
|----------|-------|----------|
| `--cream` | `#FAF7F2` | Page background |
| `--ink` | `#1C1917` | Primary text, table headers |
| `--sage` | `#5C7A5A` | Primary accent (active tabs, buttons, GC contribution color) |
| `--sage-light` | `#E8F0E7` | Hover states, accent card backgrounds |
| `--gold` | `#B8860B` | Secondary accent (match events, gold metric cards) |
| `--gold-light` | `#FDF6E3` | Match event row backgrounds, gold card backgrounds |
| `--sky` | `#2B5F8A` | Avg contribution banner background |
| `--sky-light` | `#EAF2F8` | Sky-tinted backgrounds |
| `--border` | `#DDD8D0` | All card and table borders |
| `--match-bg` | `#D4EDDA` | Match event row highlight |
| `--match-fg` | `#1D5C35` | Match event row text |
| `--neg-bg` | `#FCE4E4` | Negative balance highlight |
| `--neg-fg` | `#8B1A1A` | Negative balance text |

**Typography:** `'Playfair Display'` (serif) for headings and large metric values; `'DM Sans'` (sans-serif) for all body text and UI elements.

**Key component classes:**
- `.card` — white rounded card with border and shadow
- `.mc` — metric card (cream bg); modifiers: `.accent` (sage), `.gold`, `.warn` (red)
- `.mc-label`, `.mc-value`, `.mc-sub` — metric card typography
- `.avg-banner` — navy-to-dark-blue gradient strip for the avg contribution display
- `.tbl-wrap` — scrollable table container
- `.match-row` — green highlight on match event rows
- `.init-row` — gold highlight on GC initial deposit rows
- `.child-btn`, `.child-btn.sel` — grandchild selector buttons

---

## Editing guidelines for Claude

### When adding a new slider/assumption
1. Add the `<input type="range">` with an `id` of `sl-{name}` and a display `<span>` with `id` of `ov-{name}` in the Assumptions panel HTML
2. Read it in `getP()` and add the key to the `p` object
3. Use `p.yourKey` in the relevant projection function
4. Update the display in `update()`: `document.getElementById('ov-{name}').textContent = ...`

### When adding a new tab
1. Add a `<button class="nav-tab">` to `<nav>` with `onclick="switchTab('newtab', this)"`
2. Add `<div id="panel-newtab" class="panel">` inside `<main>`
3. Add a `renderNewTab(p, ...)` function
4. Call it from `update()` and inside `switchTab` if needed

### When changing financial logic
- All projection logic lives in `projectGC`, `projectGPAcct`, and `buildCF1`
- `update()` is the single trigger point — it calls all projections and renders
- If you change what a projection function returns, update every `renderX` function that consumes it
- Match eligibility check: `gcMonth > 0` (always true with default $20/mo — no separate flag needed)

### Formatting helpers
```js
fmt$(v)    // formats as $X,XXX; negative as ($X,XXX)
fmtMini(v) // formats as $X,XXX (always positive, no parens)
ym(y, m)   // returns "Jan 2026" style string
addMonths(y, m, n) // returns {y, m} after adding n months
```

### Do not change
- The single-file architecture — keep everything in `index.html`
- The CDN URLs for Chart.js and Google Fonts
- The `GC_STARTS` array logic (GC 1 = Jun 2026, each +1 month)
- The GP Investment Account start date (May 2028)
- The `update()` → `renderX()` data flow pattern

---

## Known limitations / future work Grandma may request

- **Grandchild names** — currently "Grandchild 1" through "Grandchild 23"; a future edit could add a names array at the top of the script
- **Per-child overrides** — all GCs share the same assumptions; individual rate/contribution overrides are not yet implemented
- **Inflation adjustment** — projections are nominal, not real
- **Tax modeling** — not included; this is a general investment account with no tax treatment
- **Export** — no PDF or CSV export; data only visible in browser
- **Mobile** — the tables scroll horizontally on small screens; layout is responsive but dense
