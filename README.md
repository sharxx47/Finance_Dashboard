# Family Finance Dashboard — Project Handoff

Carry this into a new chat to pick up where we left off. Attach it along with `Family_Finance_Dashboard_Specifications.md` if the next task is a data update.

---

## Where things stand

The dashboard is **final and approved**, and current as of the 10 Aug 2026 workbook. It is a single self-contained HTML file with Archivo embedded, so it opens offline with no local dependencies.

On 14 Aug it gained its two interactions — a drill-down on every mosaic tile and a scrub on the chart. Both are specified in full (§5a, §6b). The savings companion page is the only thing still open.

### Files in the project

| File | What it is |
|---|---|
| `Family_Finance_Dashboard.html` | **The final dashboard.** Self-contained; the only thing that changes month to month is the `DATA` block near the top. |
| `Family_Finance_Savings.html` | **Companion page: "Where saving comes from".** Splits expenses into fixed vs flexible, with a trim-it-by calculator. Cross-linked to the main dashboard. |
| `Family_Finance_Dashboard_Specifications.md` | Exhaustive spec for rebuilding **both** pages from an updated workbook. Feed to Claude together with the xlsx. |
| `Family_Finance.xlsx` | The source workbook (shared into `uploads/`). Transaction log from June 2026 onwards; live data starts July 2026. |
| `Family Finance Flow.dc.html` | Earlier dark-purple treemap version, kept on request for possible later use. |
| `Fused B v6.html` | The drill-down exploration the final design came from. The other `Fused`/`Layout` files are the alternatives it was chosen against. |
| `Layout 1 - stacked.html`, `Layout 3 - nested.html` | The two drill-down structures rejected in favour of the fusion. |
| `Fused A`, `Fused B`, `Fused B v2`–`v5` | Iterations of the fused drill-down, kept at the user's request. |
| `Teal shades.html` | The four-way accent comparison that chose `#2DD4BF`. |
| `Interaction explorations.html`, `Drill-down depth.html` | The interaction and depth options the user picked from. |
| `versions/` | Dated copies of the dashboard from before each significant change. |
| `CLAUDE.md` | Standing project rules (iPhone-first, PKR verbatim). Applied automatically in this project. |
| `support.js` | Auto-generated runtime for `.dc.html` files. Do not edit. |

`Family_Finance_Dashboard.html` is the single source of truth for the design — edit it directly.

**Do not delete any file without asking.** Exploration files are how the user compares options, and they stay useful after a decision is made — a rejected layout is the record of why the chosen one won. Three were deleted as "superseded" and had to be rebuilt. This is now a standing rule in `CLAUDE.md`.

---

## The design, as settled

**Concept.** A phone-first, scroll-once dashboard. Frosted glass panels over a dark ember-lit backdrop, with spending shown as a treemap where block area equals money.

**Structure, top to bottom:**

1. Header — "Family Finance" + date pill
2. Hero — "LEFT TO SPEND" 64,350, then a spent/kept split bar
3. Pace pair — "Last 7 days" with a week-on-week caption, "Per day" with the day count
4. "Where it went" — the mosaic, captioned "area = amount"
5. "Day by day" — running-balance line over the last 30 days, with a per-day spend rail and dated axis beneath it

No footer and no cross-page navigation on either page — both removed at the user's request (13 Aug).

**Interactions (14 Aug).** Tapping any mosaic tile opens a detail band under its own row: the top 5 *places* paid, each unfolding into its own items, then the top 8 items bought, with a pill that filters to items from outside the top 5. Dragging across the chart reads out any day's balance and spend. Nothing else on the page is tappable.

**Colour.** Core orange `#FF5F00` (the user's favourite colour, explicitly chosen from five options). Teal `#2DD4BF` as the single cool contrast — it only ever means money in, money kept, or a balance being read. Ink `#FFF4EC` on base `#0C0603`. Mosaic uses a 9-step orange ramp, darkest = biggest spend.

The teal was `#00D6C7` until 14 Aug, when the user asked to see alternatives: four candidates were built side by side (the original, a softer teal, a cyan, a violet) and the softer `#2DD4BF` was chosen on a phone. It is defined once as `TEAL` plus four `--hl-*` tokens on `body`; changing those five changes every teal on the page.

**Backdrop.** A single ember glow from above on near-black. An earlier four-colour version was rejected for fighting with the mosaic.

**Type.** Archivo throughout, chosen by the user from a three-way comparison (against Playfair+Source Serif, and Space Grotesk). Picked for holding up bold at small sizes with tabular figures.

**Mosaic tiles.** "Amount-led" treatment (the user picked B from three options): amount large on top, category name beneath in small caps at `.72` opacity. Type auto-fits per tile using canvas text measurement — the amount gives up size before the name does.

**Category names are the workbook's own, verbatim.** "Groceries & Household", "Personal Care & Beauty", "Clothing & Accessories" — not shortened. Bigger-spend categories get bigger tiles, so the long names have room, and the per-tile fitter handles the rest: labels stay **always uppercase**, tracking tightens (`.13em → 0`) before the font size is allowed to drop, with an 8px floor. "Everything else" is the only synthesized label.

---

## Preferences learned along the way

These were expensive to discover — worth honouring rather than rediscovering.

- **Viewed almost entirely on iPhone.** Phone-first is non-negotiable, not a responsive afterthought.
- **Wants radical difference when asking for alternatives.** "Same layout, new colours and fonts" was rejected outright, twice. New directions must differ in information model, not just paint.
- **Regressions are the thing that frustrates them most.** Several times a fix silently changed something adjacent — a dim filter, a chevron, nine spacing values, a type-size rule. Copy values verbatim between files, change only what was asked, and say what was checked. All seven of their rules are in `CLAUDE.md`.
- **Wants rules written as rules, not as today's values.** "When we agree a rule, write it down as a rule that holds for future data." Every threshold in the spec is expressed as a share or a formula for this reason.
- **Will say when they want to decide from something built** rather than described. Options get compared on a phone, side by side.
- **Dislikes clutter and unexplained numbers.** A newspaper-style version was rejected for being cluttered; an early table was rejected because "440,580 / 85,467" sat under a month name without labels.
- **Dislikes text-heavy layouts.** The fix that landed was charts carrying the numbers, not more prose.
- **Likes skeuomorphic touches in principle** but rejected the heavy execution (paper grain, rubber stamp, halftone). Restraint won.
- **Wants full workbook category names**, not abbreviations, and labels in uppercase.
- Reacts well to being shown real built options side by side rather than being asked to imagine them.

---

## 13 Aug — the dashboard is final

Six changes, in order:

1. **Footer and cross-link removed**, and the leftover Google Fonts `<link>` stripped. The typeface was already embedded, so the file now makes **zero external requests** and opens with no connection at all.
2. **The biggest-day callout became a filled pill** on a short stem. It had been a small triangular caret — the only hard-cornered shape in a design built on soft radii. Chosen from four options (rounded cap, dot-and-stem, soft teardrop, pill).
3. **Pill overflow handled properly.** It had a fixed clamp that assumed a label width; it now measures the text and shifts by the smallest amount that keeps it inside the panel. The pill itself can never be overflowed — it sizes to its contents.
4. **Bar spacing fixed.** The end bars were being nudged inward to avoid straddling the plot edge, which closed the gap to their neighbours and visibly joined the first two and last two bars. The whole x-scale is now inset by half a bar width, so every bar centres on its own day.
5. **The header sub became the day count** — `18 days`, switching to `last 30 days` once the window stops growing. This is what stops the panel being read as a calendar month.
6. An intermediate version put that span in a **caption line** under the header. It worked but added 24px to an already tall panel; reverted, and the span moved to the sub instead.

The spec was audited against the file afterwards: CSS block, `buildChart`, `windowLabel`, `wowText` and every literal (200px plot, 36px rail margin, the Y formula, the dot-shrink thresholds) all verified identical.

---

## 14 Aug — the two interactions

The user asked to make the dashboard interactive, "especially when it comes with data". Four options were built and shown; they picked **tap a tile** and **drag across the chart**, dropped a who-paid summary tile as "not something we need on a shared dashboard", and set the default state as **closed and calm**.

### Deciding what a tile opens

Two depths were built with real data so the difference was visible: **who you paid** (Groceries → 15 places) versus **what you bought** (the same total → 155 items). Places won — a short list, the same names each month, so one can be watched growing. The user then asked to fuse both, and three structures were built: stacked lists, tabbed, and items nested under each place. The final design is a fusion of stacked and nested, chosen after two more rounds:

- Top 5 **places**, each unfolding up to 4 of its own items.
- Top 8 **items** below, because the biggest single purchase often comes from a place outside the top 5 — the case the nested view alone cannot show.
- A pill naming how many of those 8 came from outside the top 5, which filters to them when pressed.

### Details the user drove

- **"Places", not "vendors"** — a family does not have vendors.
- **The lift.** The opened tile scales up and its siblings scale down by the same amount, because lifting alone made it overlap its neighbours. The siblings desaturate *and* darken; brightness alone looked like a rendering fault.
- **The detail opens under its own row**, not at the bottom of the mosaic.
- **Dates are hidden until a tap asks for them** — the user's idea, and it recovered ~50px.
- **Revealed lines cascade at 35ms intervals.** The user called the un-cascaded version "clunky" next to a place tap; this fixed it.
- **The pill fills solid when pressed**, and the whole row is a 44px target.
- **Switching categories is one gesture** — collapse, move, refill, grow, sequenced on `transitionend`.
- **Amount type is ranked by amount.** It had been keyed to tile height, which printed Housing (the largest) smaller than Groceries. Now the biggest number is always the biggest type, keyed to share of total so it survives new data.

### The scrub

Press and drag anywhere on the plot: a crosshair, a dot on the line, and a pill reading the day, its balance and its spend. It exists because the chart labels three days and the other 27 carry numbers nothing had room for. `touch-action: pan-y` so a vertical drag still scrolls the page; the rail's bars restore to their own opacities on release, not to 1, so quiet days stay quiet.

### Spec work

§8 was a 418-line code listing that had gone stale twice in one day — it still described the pre-drill-down mosaic while §5a described the new one. It is now a pointer to the HTML plus the two pieces prose cannot carry (the chart's scale/path math, the row grouping). **The spec and the HTML travel together; neither is a complete brief alone.**

§9's checklist gained 27 items covering both interactions.

---

## Distribution — the constraint that decides things

**Both pages are shared over WhatsApp, one file at a time, and opened on phones.** Two consequences, both already applied:

- **The typeface is embedded as base64.** Neither file makes a single external request; they open with no network at all. Do not reintroduce a Google Fonts `<link>`.
- **The pages cannot link to each other.** A relative `<a href>` only resolves if both files sit in the same folder under exactly those names, which is not true for a WhatsApp recipient. A cross-link pill was built and removed. If the two views ever need to be reachable from each other, **merge them into one file with an in-page toggle** — one attachment, no filesystem assumptions.

---

## The companion page — `Family_Finance_Savings.html`

Built after the user said they want to start saving. The main dashboard shows **where money went**; nothing on it distinguished spending the family can change from spending they cannot, which is the only question that leads to a decision.

Every category is assigned to **fixed** (231,016 · 43%) or **flexible** (304,389 · 57%). That split is a judgement about their life, not something the workbook states — it was proposed, shown to the user, and accepted as built. Groceries is the real borderline case and currently sits in flexible.

Structure: a hero on the flexible total with a fixed/flexible split bar, a "Trim it by" 5/10/20% calculator projecting a monthly saving, then the two category lists. Both lists share one bar scale so a bar means the same thing in each panel.

**Still in progress** — the main dashboard was finalised first, on 13 Aug. This page has had no review pass yet.

No savings target is set, so the page is framed as "here is the pool you could draw from", not "here is your progress". When they set a goal, this is the page it belongs on.

---

## Aug 2026 revision — why sections 3 and 5 changed

The user raised both, and both had the same root cause: the dashboard mixed cumulative figures with monthly ones.

**In / Out went stale.** All-time in and out only restate the hero, and grow less informative every month. Replaced with pace: spending over the last 7 days against the 7 before it, and the daily average. These answer "is the gap widening or shrinking?" — which is what the user asked for, since they have **no savings target yet** (tracking only began late July 2026).

**The month comparison misled.** Salary lands as a single lump, and tracking began 24 July, mid-cycle — so July showed +355,112 and August −290,762, which reads like a catastrophe rather than one salary cycle. Replaced with a running balance, one point per day: a lump sum is a step up, spending is the slide down, no net figures at all.

The daily series makes the shape plain: balance peaked at **431,610** after July's salary on the 27th, slid to a low of **27,186** by 6 August, then August's salary landed the next day. Spending is currently easing — 247,522 over the last 7 days against 278,912 the week before, down 11%.

No savings goal is on the dashboard by design. Add a goal line when the family sets one.

**The peak callout was then anchored to its bar.** A label floating above the rail spans six or seven bars and names none of them. The biggest day is now identified three ways at once: the bar is tinted `#FF8A3D` at full opacity, a caret sits on its top edge, and the label is centred over it.

**Then the panel gained a spend rail and a rolling window.** The user picked the rail from three x-axis variations (dated 5-day ticks; a dot per day; the rail) and asked for a dashed average line and a date-and-amount callout on the biggest day. They also set a standing constraint: **the dashboard is updated about every 10 days and must still make sense at the end of the month, with no information that goes stale.** That is why the panel and the "Per day" tile now measure a rolling **30-day window** rather than all history — the bars never thin out, and the average, biggest day and lowest point never freeze on an old date. The hero keeps the all-time position. `WIN` is one constant in the file.

Axis dates run on an exact 5-day step anchored to the as-of date and walked backwards, so the newest end always lands on a tick.

---

## Data currently in the dashboard (Aug 2026 snapshot)

All PKR, verbatim from `Family_Finance.xlsx`, as of 10 Aug 2026 (333 transactions).

```
Totals      in 599,755   out 535,405   left 64,350   kept 10.7%
Tracked     24 Jul – 10 Aug 2026, 18 days
Pace        last 7 days 247,522 vs prior 7 278,912 (−11%)   per day 29,745
Balance     peak 431,610 (27 Jul)   low 27,186 (6 Aug)   ends 64,352
Biggest day 112,647 on 5 Aug
Fixed       231,016 (43%)   flexible 304,389 (57%)

Housing                 123,343
Groceries & Household    91,422
Personal Care & Beauty   84,473
Utilities & Bills        64,298
Dining & Takeout         48,111
Clothing & Accessories   39,868
Education                18,490
Domestic Staff           17,500
Everything else          47,900   (derived: totalOut − listed)
```

"Everything else" is the tail below rank 8: Entertainment 13,485 · Transport 12,306 · Allowances & Pocket Money 10,000 · Subscriptions 7,385 · Medical & Health 3,882 · Charity & Sadqa 700 · Bank, FX & Tax Charges 141.

Transfers (4,098 in August) are excluded from both income and expenses.

### Two known quirks in the workbook

The source rounds, so recomputing drifts by a rupee:

- Daily incomes **sum to 599,756**, but the stated total is **599,755**.
- Daily expenses **sum to 535,404**, but the stated total is **535,405**.
- Both drifts push the same way, so the running balance ends at **64,352** while the hero shows **64,350** (599,756 − 535,404 against 599,755 − 535,405). Both are correct: the hero uses workbook totals, the chart accumulates daily rows. Do not reconcile by editing a daily figure.

**Always take stated figures from the workbook; derive only what it doesn't state.** The spec encodes this, and the dashboard's `DATA` block accepts optional `totalIn` and `totalOut` fields for it.

### The one structural gotcha

The main dashboard **paints static pre-rendered markup first, then re-renders the same content from JS**. Every change to the chart, the tiles or the page structure has to be made in **both** copies — the markup inside `<main>` and the template string in `render()` — or it looks right on load and vanishes the moment anything re-renders. This has bitten twice.

### Reading the workbook

The Dashboard, Category Summary and Charts tabs are **formulas with no cached values** — reading them straight out of the file returns zeros. Compute from the `📋 Transactions` tab instead: column A date, D category, E amount. Rows whose category is Income or Transfer are not expenses, and **income rows are stored negative** — take the absolute value.

---

## Updating with new data

Best path: start a new chat, attach `Family_Finance_Dashboard_Specifications.md` and the updated `Family_Finance.xlsx`, and say "rebuild my dashboard from this spec."

The spec covers source data and exact category naming, every derived formula, the full design system, the mosaic algorithm including auto-fitting, update rules for new months and categories, and a self-contained reference implementation where only the `DATA` block changes.

The `DATA.daily` array is the thing that grows: one row per calendar day, quiet days included. Past roughly 120 days the spec says to switch it to one row per week rather than truncate — the running balance should stay unbroken.

---

## Loose ends

- **The savings companion page is unreviewed.** It has had no pass since it was built, and the spec deliberately says nothing about it yet — the user asked to hold that until it is finalised.
- **The 30-day window has not been seen doing its job.** History is 18 days, so nothing is being trimmed yet. Worth a look at the next update.
- `versions/` holds dated copies of the dashboard from before each significant change on 13 Aug. Earlier iterations were not kept — that was a miss, and copies are now taken before each change.
- The fixed/flexible split is a judgement call, not workbook data. If the family disagrees with any assignment, move the category between the two arrays in `DATA` and the whole companion page re-cuts itself.
- The window is 30 days and the history is 18, so the panel currently shows everything. The rolling behaviour only becomes visible after 30 days of data — worth a look at the next update.
- Deferred: a savings target. The user has none yet, so the dashboard measures direction instead of distance-to-goal. Revisit once they set one — a goal line on the chart is the natural place.
- Possible next steps, none discussed in detail: a per-person breakdown (the workbook's "For" tag — Salah, Zaydan, Girls, Kids, Saman, Shahzeb, Household, Gift — already supports it), budget vs actual, a savings goal tracker, or a month filter.
