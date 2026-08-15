# Family Finance Dashboard — Rebuild Spec

Give this file to Claude along with your updated `Family_Finance.xlsx` and say:

> Rebuild my family finance dashboard from this spec using the numbers in the attached workbook. Output one self-contained HTML file.

Everything Claude needs is below: the numbers to pull, the formulas, the full design system, and a working reference implementation. The only thing that changes between updates is the `DATA` block at the top of the HTML — the family refreshes it roughly every 10 days.

---

## 1. Source data

Pull these from the workbook. Nothing else is needed.

| What | Where | Example (Aug 2026) |
|---|---|---|
| As-of date | Latest transaction date | `10 Aug 2026` |
| Daily in | Sum of income rows per **day** | 27 Jul 440,580 · 7 Aug 159,176 |
| Daily out | Sum of expense rows per **day** | 5 Aug 112,647 … |
| Workbook totals | Stated income / expense totals | 599,755 · 535,405 |
| Category totals | Expenses grouped by category, all months | Housing 123,343 … |

Rules:

- **Figures are PKR.** Keep them verbatim from the workbook — never round, estimate, or "clean up" a number.
- **The unit of time is the day, not the month.** Tracking began 24 July 2026, mid-cycle, and salary lands as one or two lumps a month, so calendar months are not comparable to each other. Build the `daily` array from the transaction rows: one entry per calendar day from the first transaction to the as-of date, **including days with no activity** (they are what makes a flat stretch read as flat).
- **Income rows are stored negative** in the workbook's amount column. Take the absolute value.
- **Transfers are excluded** from both in and out — they move money between the family's own accounts.
- **Category totals are cumulative across all months**, not per-month.
- List the **top 8 categories by amount, descending.** Everything below rank 8 collapses into one bucket called "Everything else" (see §6).
- **Use the EXACT workbook category names on the tiles**, verbatim (the family's preference) — e.g. "Groceries & Household", "Personal Care & Beauty", "Clothing & Accessories". Do not shorten or rename them. This works because bigger-spend categories get bigger tiles, so the longer names have room; the per-tile auto-fit (§5) shrinks / wraps as needed. Only "Everything else" is a synthesized label (see §6). (If a long name ever clips in a genuinely small tile, shorten just that one — but full names are the default.)

### Current values (Aug 2026 snapshot)

```
Daily  [label, in, out]        24 Jul → 10 Aug, 18 days
  24 Jul       0        510      3 Aug        0     38,196
  25 Jul       0          0      4 Aug        0      1,708
  26 Jul       0      6,160      5 Aug        0    112,647
  27 Jul 440,580      2,300      6 Aug        0     11,157
  28 Jul       0      1,480      7 Aug  159,176     49,681
  29 Jul       0     42,261      8 Aug        0      3,861
  30 Jul       0      8,682      9 Aug        0     12,694
  31 Jul       0     24,074     10 Aug        0     55,774
   1 Aug       0     90,960
   2 Aug       0     73,259

Categories (cumulative)
  Housing                  123,343
  Groceries & Household     91,422
  Personal Care & Beauty    84,473
  Utilities & Bills         64,298
  Dining & Takeout          48,111
  Clothing & Accessories    39,868
  Education                 18,490
  Domestic Staff            17,500
  Everything else           47,900   ← derived, see §6
```

---

## 2. Derived values — exact formulas

| Value | Formula | Current |
|---|---|---|
| Total in | workbook total, else Σ daily in | 599,755 |
| Total out | workbook total, else Σ daily out | 535,405 |
| **Left to spend** | totalIn − totalOut | **64,350** |
| Kept % | left ÷ totalIn × 100 | 10.7% |
| Spent % | 100 − kept% | 89.3% |
| Last 7 days | Σ out over the final 7 entries | 247,522 |
| Prior 7 days | Σ out over the 7 before those | 278,912 |
| Week on week | (last7 − prev7) ÷ prev7 × 100 | −11% |
| Per day | Σ out ÷ days, **over the window** | 29,745 |
| Running balance | cumulative Σ (in − out), day by day | ends 64,352 |
| Lowest point | min balance in window, **after the first money-in** | 27,186 on 6 Aug |
| Biggest day | max out in window | 112,647 on 5 Aug |

### The rolling window

**The "Day by day" panel and the "Per day" tile show the last 30 days, not all history.** The running balance accumulates over the full `daily` array; only the *view* is trimmed:

```js
const WIN  = 30;
const view = series.slice(-WIN);          // balance values stay absolute
```

This exists because the family updates the dashboard roughly every 10 days. On an all-history view the bars would thin to a smear within a couple of months, and the average, the biggest day and the lowest point would all freeze on dates from months ago. Every figure in the panel is measured over the same window, so the panel still says something true at the end of every month. **The hero keeps the all-time position; the panel answers "how are we doing lately".**

With fewer than 30 days recorded the window is simply everything, so nothing looks different yet — the tile caption prints the real span (`18-day average` today, `30-day average` once there is more history).

> **Take every stated figure from the workbook rather than recomputing it.** The source rounds, so arithmetic on its numbers can drift by a rupee. Both sides drift in the Aug 2026 data: the daily rows sum to 599,756 in and 535,404 out, while the workbook states 599,755 and 535,405. The workbook wins, every time. The reference implementation takes optional `totalIn` and `totalOut` fields for exactly this — supply them whenever the workbook states the figure, and let the code derive only what the workbook does not.

The running balance ends at 64,352 while the hero shows 64,350. Both drifts above push the same way, so they add: 599,756 − 535,404 = 64,352 from the daily rows, against 599,755 − 535,405 = 64,350 from the stated totals. This is expected and correct — the hero takes the workbook's totals, the chart accumulates daily rows. Never reconcile them by editing a daily figure.

**The lowest point ignores everything before the first money-in.** Tracking began 24 July, three days before July's salary, so the balance opens negative — an artefact of when recording started, not a debt. Anchoring the low to the post-salary stretch is what makes it meaningful.

Display rounding: the **split bar uses the exact percentages** (89.3 / 10.7) for its widths, but the **labels under it show whole numbers** ("89% spent", "11% kept"). Everything else displays as a thousands-separated integer.

The chart's vertical scale spans the running balance's own range, **always including zero**, with 8% headroom top and bottom:

```
bmax = max(balance, 0) = 431,610   (27 Jul, just after salary)
bmin = min(balance, 0) =  −6,670   (26 Jul, before the first salary)
y(v) = 8 + (bmax − v) / (bmax − bmin) × 84      // percent from top
```

Zero always gets a hairline rule, so a balance heading below it is visible as it happens.

Colour: **the accent teal (§3) for the balance line and money-in markers, orange `#FF5F00` for the low point.** Negative figures use a true minus sign `−` (U+2212), not a hyphen.

---

## 3. Design system

### Colour

| Token | Value | Used for |
|---|---|---|
| Core orange | `#FF5F00` | Spend, overspend, mosaic rank 1 |
| Teal accent | `#2DD4BF` | Money in, kept, positive nets, the balance line, salary dots, the outside-top-5 highlight |
| Accent ink | `#04302D` | Text on a filled teal pill |
| Bar tint | `#FF8A3D` | Biggest-day bar and pill, the lit-place highlight |
| Ink | `#FFF4EC` | All text on dark |
| Ink muted | `rgba(255,244,236,.5)` → `.72` | Labels, captions |
| Tile ink (dark) | `#2A1206` | Text on light orange tiles |
| Page base | `#0C0603` | Body background |

**Backdrop** — fixed layer behind everything, single ember glow from above on near-black. No other colour blobs:

```css
position: fixed; inset: 0; pointer-events: none;
background:
  radial-gradient(120% 68% at 50% -12%, rgba(255,95,0,.60) 0%, rgba(255,95,0,.16) 38%, rgba(255,95,0,0) 66%),
  radial-gradient(90% 50% at 50% 108%, rgba(255,120,20,.16) 0%, rgba(255,120,20,0) 60%),
  linear-gradient(180deg, #17100C 0%, #0E0907 46%, #090605 100%);
```

**Mosaic palette** — a 9-step orange ramp, darkest = biggest spend:

```
0 #FF5F00   3 #FF9F5E   6 #FFD3B2
1 #FF7524   4 #FFB27B   7 #FFE1CB
2 #FF8B42   5 #FFC397   8 #FFEDDF
```

Text on tiles: **white on steps 0–1, `#2A1206` on steps 2–8.**

**The teal is `#2DD4BF`** — chosen from four candidates (the original `#00D6C7`, this softer teal, a cyan, and a violet) by building all four and comparing them on a phone. The earlier `#00D6C7` sat too close to pure cyan and vibrated against the orange at small sizes. Whatever the accent is, it must be **one value used everywhere** — the balance line, the salary dots, the kept bar and the drill-down's outside-top-5 highlight all read it from the same place. Changing it means changing it in all of them, not some.

Accent derivatives live as tokens on `body` so the drill-down never hardcodes them:

```css
body{--hl:#2DD4BF;--hl-ink:#04302D;--hl-line:rgba(45,212,191,.34);--hl-fill:rgba(45,212,191,.09)}
```

`--hl-line` and `--hl-fill` are the same teal at .34 and .09 alpha. If the accent changes, all four change together.

**The accent is named here and nowhere else.** Everywhere the balance line, money-in markers, day dots, the "kept" half of the split bar, the 7-day tile's delta, the scrub dot and the scrub pill's balance figure are described, they mean this value. In the file it is the `TEAL` constant plus these four tokens — changing those five changes every one of them.

### Type

**Archivo** throughout, weights 400/500/600/700. It is a grotesque built for headlines and tables — heavy weights stay solid at small sizes and its tabular figures hold their column.

**The font is embedded in the file as a base64 `@font-face`, not linked from Google Fonts.** The dashboard is shared over WhatsApp and opened on phones that may be offline or on a slow connection, so a network request for the typeface is a single point of failure for the whole design. A rebuild must keep the embedded `@font-face` block and must not reintroduce `<link>` tags to fonts.googleapis.com. The finished file has **zero external references**.

| Element | Size | Weight | Tracking |
|---|---|---|---|
| Hero number | `clamp(54px,16vw,66px)` | 700 | −.045em |
| Tile figures | 36 / 30 / 23 / 20px by share | 700 | −.035em |
| Section titles | 21px | 700 | −.015em |
| Wordmark | 19px | 700 | −.015em |
| Chart markers | 11px | 600 | .07em |
| Axis dates | 11px | 500 | — |
| Tile captions | 11px | 500 | — |
| Rail callout | 10.5px | 600 | .05em |
| Rail average | 9.5px | 600 | .09em |
| Body / captions | 12–13px | 500 | — |
| Uppercase labels | 12–13px | 500 | .12–.14em |

All figures use `font-variant-numeric: tabular-nums`.

### Glass panels

Every card is the same recipe, varying only in radius and blur strength:

```css
background: rgba(255,255,255,.08);          /* hero uses .09 */
border: 1px solid rgba(255,255,255,.16);    /* hero uses .18 */
border-radius: 24px;                        /* hero 30px, wide panels 28px */
-webkit-backdrop-filter: blur(24px) saturate(160%);
backdrop-filter: blur(24px) saturate(160%);
box-shadow: inset 0 1px 0 rgba(255,255,255,.18);
```

The hero panel adds a drop shadow: `0 20px 50px rgba(0,0,0,.38)`, and uses `blur(26px) saturate(170%)`.

**The `-webkit-` prefix is required** — without it the glass does not frost on iOS Safari.

### Layout — phone first

This dashboard is viewed almost entirely on an iPhone. Desktop is just the same column, centred.

```css
max-width: 430px;
margin: 0 auto;
padding: calc(20px + env(safe-area-inset-top)) 16px calc(34px + env(safe-area-inset-bottom));
display: flex; flex-direction: column; gap: 14px;
```

Also required: `min-height:100dvh` on the root, `overflow-x:hidden` on body, `-webkit-text-size-adjust:100%` on html, and `viewport-fit=cover` in the viewport meta. No horizontal scrolling anywhere.

---

## 4. Page structure

Top to bottom, one column:

1. **Header row** — "Family Finance" left, as-of date right in a small glass pill.
2. **Hero panel** — "LEFT TO SPEND" label, the big number with "PKR" beside it, then the spent/kept split bar with its two legend items.
3. **Pace pair** — two equal glass tiles side by side: "Last 7 days" with its week-on-week caption, and "Per day" with the day count. Uppercase label, 28px figure, 11px caption.
4. **"Where it went"** — the mosaic (§5), captioned "area = amount".
5. **"Day by day"** — the running-balance line over the rolling window, subtitled with the span it covers (§6a), with the "Spent per day" rail and dated axis beneath it. **No legend** — every element is labelled where it sits.

**The page ends there — no footer, no source line, no navigation.** The family knows where their own numbers come from, and a provenance line is the sort of thing that reads as boilerplate on a phone. Removed at the user's request; do not reinstate it.

---

## 5. The mosaic — where it went

A treemap where **block area equals money spent**. This is the centrepiece; get it right.

### Rows

Categories sorted descending, then grouped:

- Rank 1 alone on its own full-width row
- Ranks 2–3 share a row, 4–5 share a row, 6–7 share a row
- Rank 8 alone, "Everything else" alone

With the current data: `[[0], [1,2], [3,4], [5,6], [7], [8]]`

### Row height

```
height = max(66, round(rowTotal / totalOut × 660))
```

660px is the scale factor for the whole mosaic; 66px is a floor so a two-line label always fits. Rows above the floor stay strictly proportional.

### Tile width

Within a row, each tile takes its share of the row's total:

```
width = calc(<share>% - <gap allowance>px)
gap between tiles = 4px, so a 2-tile row deducts 2px from each
```

### Tile content — "amount-led"

Flex column-**reverse**, so the amount sits on top and the name reads as a caption beneath it:

```css
display:flex; flex-direction:column-reverse;
align-items:flex-start; justify-content:flex-end;
padding:13px 14px; border-radius:14px;
gap: 8px;   /* 5px when the row is under 130px tall */
min-width:0; overflow:hidden;
```

- **Amount** — weight 700, `letter-spacing:-.035em`, `line-height:1.05`, `white-space:nowrap`, tabular figures.
- **Name** — weight 600, `opacity:.72`, `line-height:1.15`, `overflow-wrap:anywhere`, uppercase with `.13em` tracking.

### The mosaic exists twice — change both copies

The page paints **static pre-rendered markup** inside `<main>`, then `render()` rebuilds the same content from `DATA`. Every change to the tiles, the chart or the page structure has to be made in **both** copies, or the design is briefly wrong on first paint and corrects itself a frame later.

This is not cosmetic. When the amount type-size rule changed, only `render()` was updated, so the page painted with Groceries as the biggest number and snapped to Housing when the script ran — five of nine tiles wrong, in the one moment a reader is looking at the mosaic for the first time.

**After any change to the tiles, verify the static copy produces the same sizes, colours and order as `render()` does.** The static copy assumes a 318px mosaic width (iPhone 13+) and is the only place a literal value is allowed to be written by hand.

The two copies are deliberately not identical in one respect: the static tiles are `<div>`s with no `data-cat`, since nothing can be tapped before the script runs. Keep that; it is the interactivity, not the appearance, that differs.

### Auto-fitting (the part that breaks if you skip it)

Tiles vary enormously in size, so type must be sized per tile or long names clip. Two fitters:

**Measure the text, do not estimate it.** Use a canvas 2D context with the real font string — character-width averages are not accurate enough for Archivo's uppercase, and a tile that is 4px too narrow wraps a word mid-letter.

```js
const ctx = document.createElement("canvas").getContext("2d");
const textW = (text, size, weight, track) => {
  ctx.font = weight + " " + size + "px Archivo, sans-serif";
  return ctx.measureText(text).width + text.length * size * track;
};
```

**Amount size — driven by the amount, never by the tile.** The biggest number on the mosaic must be the biggest number set. This is not automatic: a row's height comes from the row's **combined** total, so a tile sharing a tall row can be physically taller than a larger tile sitting alone. Sizing from tile height therefore inverts the order — it printed Housing (23% of spend, the largest) at 30px while Groceries (17%) got 36px.

The rule, in three parts, all of which are load-bearing:

```js
// 1. tiers from each category's SHARE OF TOTAL SPEND, not its rank and not the tile
base = share >= .18 ? 36 : share >= .12 ? 30 : share >= .07 ? 23 : 20;
// 2. walk categories in DESCENDING amount, capping each size by the one above it
//    cap = Math.min(base, previousFittedSize)
// 3. then shrink to fit as usual: -.5 while textW(value, size, 700, -.035) > boxWidth, floor 15
```

- **Shares, not ranks.** Ranks would move a category a whole size step just because another category passed it. Shares only move it when its own share of spending actually crosses a threshold, so the steps stay stable month to month.
- **The descending cap is required.** A large amount in a narrow tile still has to shrink to fit; without the cap, that shrink can take it below a smaller neighbour and re-invert the very order this rule exists to protect. Cap first, then fit.
- **Geometry is computed before type.** Two passes: first every tile's height and width, then every tile's type size. They cannot be interleaved, because the cap needs the previous tile's *fitted* size.

**Name size** — labels are **ALWAYS UPPERCASE** (family preference — never sentence case). The longest unbreakable word must fit on one line, so to make it fit: first tighten the letter-spacing (`.13em → 0`), and only then shrink the font (8px floor). Tightening tracking first keeps a tight tile's label close to full size instead of tiny.

```js
base = 11;   // uniform label size across all tiles
// keep UPPERCASE always. try tracks .13 -> .08 -> .04 -> 0 at `base`; use the
// first that fits. if none fit even at 0 tracking, shrink the font (floor 8) at
// tracking 0. Never switch to sentence case.
// Mosaic width is measured from the real container (iPhone 13+ ≈ 318px); the
// static pre-render assumes 318 so labels stay a uniform 11px on iPhone 13+.
```

`boxWidth` = tile width − 28px (the horizontal padding). Measure the mosaic container's real rendered width and derive tile widths from it — do not hardcode a column width, or the fitting is wrong on a different phone. Re-run the fit on resize and after `document.fonts.ready`, since metrics change once Archivo actually loads.

**The amount gives up size before the name does.** A label below 11px is unreadable on a phone; a slightly smaller number is fine.

---

## 5a. The drill-down — tapping a tile

Every mosaic tile is a button. Tapping it opens a detail band **directly under its own row**, pushing the rows below down; tapping it again closes it. It exists because the mosaic answers "where did it go?" but not "who did we pay?"

There are exactly two interactions on the dashboard: this, and scrubbing the chart (§6b). Both are taps or drags on the thing being read — there is no chrome, no filter and no control anywhere.

Opening one category closes any other. Only one band exists in the DOM at a time.

### What the band contains

Two lists, one above the other, separated by a 1px rule:

1. **Who you paid** — the **top 5** places by amount, each with a count (`12×`), the amount, and a proportional bar. Bars are scaled to the largest place in *this* category, not to the category total, so the shape of the top 5 is always legible.
2. **What you bought** — the **top 8** items by amount, ranked 1–8, each with its own repeat count where it appears more than once.

**Vocabulary: "places", never "vendors".** This is a family dashboard; "vendor" is a business word. The header reads *Who you paid*, the count reads *places*.

### Counts in the group headers

The right-hand count only says "top N of M" when something is actually hidden:

```js
total <= shown  ->  "20 places"          // nothing truncated, plain count
total >  shown  ->  "top 5 of 20 places"
total === 1     ->  "1 place"            // singular
```

A category with 3 places must not claim "top 5 of 3".

### Places expand into their own items

Each place in the top 5 is itself tappable and unfolds up to **4 of its own items**, newest date shown beside each. This is the fusion that makes the band worth opening: the top-5 list answers *who*, and the unfold answers *what, from them*.

**One unfold at a time.** Opening a second place closes the first. Two open unfolds would light two different sets of items in the list below, and the highlight would be meaningless.

A place with no item detail is not tappable and gets no chevron.

### Two highlight modes

Tapping something in the band **lights the matching rows** in *What you bought*, dimming the rest to `opacity:.34`. There are exactly two modes, told apart by colour:

| Tap | Class | Lights | Colour |
|---|---|---|---|
| A place in the top 5 | `.lit-place` | Items bought from that place | `#FF8A3D` (bar tint) |
| The outside-top-5 line | `.lit-flag` | Items from places ranked 6+ | `var(--hl)` (accent) |

The two colours are the whole point: orange means "this came from the place you tapped", teal means "this came from somewhere outside the list above". They are mutually exclusive — entering either mode clears the other, and clears any open unfold.

### Reveal on tap

Each item's second line — `date · place · #rank` — is **hidden until that item is lit**. Nothing shows a date until a tap asks for it.

```css
.ivend{max-height:0;opacity:0;overflow:hidden;line-height:1.6;
  font-size:10.5px;font-weight:500;color:rgba(255,244,236,.42);letter-spacing:.02em;
  transition:max-height .22s cubic-bezier(.4,0,.2,1),opacity .18s ease,color .2s}
.irow.lit-place .ivend, .irow.lit-flag .ivend{max-height:40px;opacity:1}
```

**`max-height`, not `display:none`.** A collapsed row must still measure 0 and must still animate. A `display:none` row cannot transition; a row with any residual height puts the band's height arithmetic out by that much on every row.

The revealed lines **cascade**: `transitionDelay = index × 35ms`. Eight rows revealing at once reads as a group unfolding rather than a block appearing. This is what fixed the "clunky" feel of the outside-top-5 line, which lights several rows at once where a place tap usually lights one or two.

### The outside-top-5 line

A full-width pill below the item list: *"4 of these came from places outside the top 5"*. It is a real button — `min-height:44px`, centred text, no leading dot — and when pressed it fills solid (`background:var(--hl)`, `color:var(--hl-ink)`) rather than merely changing text colour, so its pressed state is unmistakable on a phone.

It appears **only when the count is non-zero**. Its number is computed, never written down.

### Motion

Every value here was tuned by eye on a phone. Keep them.

| Element | Duration | Easing |
|---|---|---|
| Band open (height) | .28s | `cubic-bezier(.4,0,.2,1)` |
| Band close / switch | **.17s** | same |
| Band contents fade | .16s | ease |
| Tile lift (transform) | .16s | `cubic-bezier(.4,0,.2,1)` |
| Tile fade / shadow | .2s | linear |
| Place unfold | .24s | `cubic-bezier(.4,0,.2,1)` |
| Sub-line reveal | .22s + 35ms × i | `cubic-bezier(.4,0,.2,1)` |
| Pill press | .18s | linear |

**Exit is faster than entry (.17s vs .28s).** Standard motion asymmetry — a close that takes as long as an open feels sluggish.

**Switching categories is one gesture, not two.** Tapping a different tile while a band is open runs: collapse the open band (.17s) → move the element under the new row → refill it → grow it (.28s). The two halves are sequenced on `transitionend`, with a 220ms `setTimeout` fallback in case the event never fires. Without the sequencing the band appeared to teleport.

### The lift

```css
.mosaic.picked .cell{filter:saturate(.35) brightness(.62);transform:scale(.965)}
.mosaic.picked .cell[aria-expanded="true"]{filter:none;z-index:2;transform:scale(1.035);
  box-shadow:0 16px 38px rgba(0,0,0,.55),0 2px 6px rgba(0,0,0,.35),
    inset 0 1px 0 rgba(255,255,255,.22)}
```

Three things make this work together, and dropping any one breaks it:

- **The siblings shrink by roughly what the opened tile grows** (.965 down, 1.035 up). Lifting alone made the opened tile overlap its neighbours and read as a layout error.
- **The fade is `saturate(.35) brightness(.62)`, not brightness alone.** Desaturating is what makes the unopened tiles read as *faded back*; brightness alone just makes them dark orange, which looks like a rendering fault rather than a deliberate state. This exact pair was lost once in a port and had to be restored — copy it verbatim.
- **The shadow does the depth.** Scale alone at these values is too subtle to read on a phone.

### Height arithmetic — the part that silently drifts

The band is a fixed-height element animating between measured heights. Two rules:

1. **Deltas move it; the DOM corrects it.** When a reveal or an unfold changes the content, the height is nudged by the measured delta so the motion starts at the right speed, then **re-synced from `scrollHeight` on the final `transitionend`**. Deltas alone accumulate error — a hidden `.ivend` and a revealed one do not report the same `scrollHeight`, and repeated toggling gained about 4px a cycle before the re-sync was added.
2. **A resize rebuilds the panel, so an open band is restored without animation** — measured, applied with `transition:none`, then transitions restored on the next frame. Otherwise every rotation replays the open animation.

---

## 6. "Everything else"

The mosaic must always account for **100% of total spending**, or the areas lie.

```
rest = totalOut − Σ(listed categories)
if (rest > 0) append { name: "Everything else", amount: rest, colour: #FFEDDF }
```

Currently 535,405 − 487,505 = **47,900**. If you ever list every category and `rest` is 0, omit the tile.

---

## 6a. The day-by-day panel

A filled line of the **running balance** over the last 30 days, with a **spend rail** beneath it on the same x-scale. It replaced a paired-bar month comparison, which failed for two reasons worth remembering: the net figures under each month confused more than they explained, and because salary lands in a single lump, a calendar month is not a fair unit to compare against another one. A running balance sidesteps both — a lump sum is simply a step up, and spending is the slide back down.

### One inset x-scale for everything

Every element in the panel — line, dots, gridlines, bars, ticks — is positioned by the same `X(i)`, **inset at each end by half a bar width**:

```js
const barW = (100 / (n - 1)) * 0.66;
const pad  = barW / 2;
const X = i => pad + i / (n - 1) * (100 - 2 * pad);
```

The inset is what makes bar spacing uniform. On a flush 0–100 scale the end bars would straddle the plot edge, so they have to be nudged inward — which closes the gap to their neighbours and **visibly joins the first two and last two bars**. Insetting the scale instead lets every bar centre on its own day, at the cost of a couple of percent of width that nobody can see. The area fill closes to `X(0)` and `X(n-1)` rather than 0 and 100, for the same reason.

Axis date labels centre on their tick with no edge correction: the inset already keeps them clear, and the panel's 22px padding absorbs the overhang.

### The balance line

```js
let run = 0;
const series = DATA.daily.map(([label, inn, out]) => ({ label, in: inn, out, bal: (run += inn - out) }));
const view   = series.slice(-WIN);   // WIN = 30
```

- **Line + area.** 2px teal stroke with `vector-effect="non-scaling-stroke"` so it stays 2px under the SVG's non-uniform `preserveAspectRatio="none"` scaling. Beneath it, a teal gradient fill from `.30` opacity down to `0`.
- **Vertical scale** spans the window's own balance range, **always including zero**: `y(v) = 15 + (bmax − v) / (bmax − bmin) × 72`. The 15% top and 13% bottom bands are not decoration — they are the room the marker labels occupy (see label placement below). Zero always gets a 1px `rgba(255,244,236,.20)` rule labelled "0", so a balance heading below it is visible as it happens.
- **A dot on every day.** Small teal dots on the ordinary days, so the line reads as a countable series rather than a smooth curve — you can see which single day a steep segment belongs to, and the rail bar directly beneath it gives the cause. **The dots shrink as the window fills** (5px up to 20 days, 4px to 26, 3.5px beyond) so they stay separate at 375px instead of beading into a thick line.
- **Every day keeps its dot.** Nothing is suppressed, at any window length — see the label placement below, which is what makes that possible.
- **Money-in markers.** A 9px teal dot on every day with income, labelled `+440,580`. Each dot carries a `0 0 0 3.5px rgba(12,6,3,.6)` ring so it separates from the line. Days that carry a big marker get no small dot.
- **Gridlines on the interior 5-day ticks.** 1px `rgba(255,244,236,.10)`, emitted **before** the `<svg>` so the line and fill paint over them. They tie each dot to a date on the axis below; without them the dots are countable but not placeable. Kept deliberately faint — they should be findable, not read.
- **Low marker.** One orange dot at the lowest post-money-in balance in the window, labelled `LOW 27,186`.
- **Labels get their own vertical band.** A money-in label sits **above** its dot, the low label **below** its dot — never beside it. Both are safe placements by construction: money-in is always a local peak with empty plot above it, the low is always a trough with empty plot below.

  This replaced side-by-side placement, and the reason is worth keeping. A label is 70–80px wide; day spacing is ~20px at 18 days and ~10px at a full 30-day window. Placed alongside the line, a label lies across its neighbouring days, so either a teal dot renders inside `27,186` or the dots underneath have to be dropped — and the dots that get dropped are precisely the ones on the steep runs, which is where they are worth having. Its own band costs nothing and holds at any window length.

  Horizontally the label is centred on its dot, clamped at the edges (`translateX(0)` under 12%, `-100%` over 88%) so it cannot run off. The plot is **200px** tall with `y(v) = 15 + … × 72` rather than `8 + … × 84`, and the extra headroom top and bottom is exactly the room those two bands need.
- **Label legibility.** Markers sit on top of the line and fill, so each label carries a triple `text-shadow` in the page base colour (`0 0 4px #0C0603, 0 0 8px, 0 0 12px`) to punch it out of whatever is behind it.

Plot height is 200px. The SVG uses `viewBox="0 0 100 100"` with `preserveAspectRatio="none"`; markers, labels, rail and axis are absolutely positioned in percentages over it, not inside it, so their type never distorts.

### The spend rail

Captioned "Spent per day", 46px tall, directly under the plot and sharing its x-scale — **a tall bar sits under the drop it caused**, which is the whole reason the two are stacked rather than shown side by side.

- One orange bar per day, `(100 / (n−1)) × 0.66` wide, height proportional to the window's biggest day. **Every bar is centred on its day** (`translateX(-50%)`, no exceptions) — the inset x-scale is what keeps the end ones inside the box.
- **Two rules make small days readable, and both are deliberate.** A day with **no spending at all** still draws a 1px stub at `.18` opacity, so a quiet day reads as "recorded, spent nothing" rather than as a gap that might mean missing data. A day with a **very small amount** clamps to the same 1px floor at full opacity — 510 against a 112,647 peak is 0.45% and would otherwise render as nothing. At 1px the anti-aliasing makes both look washed out, which is why the first days of a series can appear faded; that is the floor, not a third colour.
- **Average line.** A 1px dashed `rgba(255,244,236,.34)` rule across the rail at the window's daily average. The figure itself lives at the right of the "Spent per day" caption row **above** the rail, never inside it — pinned inside on the right it would sit exactly where the newest bars land, and every update would push fresh bars under it. It is the same figure as the "Per day" tile, deliberately: the tile gives the number, the rail shows which days beat it.
- **Biggest-day callout.** Three things identify the same day together: the bar is drawn in the lighter orange `#FF8A3D` at full opacity, a 1.5px stem rises from its top edge, and the date and amount sit in a **filled `#FF8A3D` pill centred over that bar**, dark text `#2A1206` on light — the same text-on-light rule the mosaic uses. The pill is measured and nudged only as far as it must be (below) while the stem keeps pointing, so attribution survives a peak at either end. A label merely floating above the rail is not enough — it spans six or seven bars and names none of them.

  **The rail carries `margin-top:36px`, and that figure is load-bearing.** The pill is ~21px tall and sits 7px above the rail; the margin is what keeps it clear of the "avg" figure at the right of the caption row. Shrink the margin and the pill crops that number whenever the peak falls in the right half.

  **The pill grows with its text — it cannot be overflowed.** `white-space:nowrap` with symmetric padding means a longer date or a seven-figure amount simply makes the pill wider; the text is never clipped. What has to be guarded is the *pill* crossing the panel edge, so it is measured with the same canvas helper the mosaic uses and shifted by the smallest amount that keeps it inside:

  ```js
  const pillW = textW(pkText, 10.5, 600, .04) + 20;   // 10px padding each side
  const pkCx  = pkX / 100 * plotW;
  let dx = -pillW / 2;                                // centred by default
  if (pkCx + dx < 0)              dx = -pkCx;                    // flush left
  if (pkCx + dx + pillW > plotW)  dx = plotW - pillW - pkCx;     // flush right
  ```

  This replaced a fixed `translateX` clamp at 15% / 85%, which assumed a label width and broke as soon as the text grew. **The stem stays anchored on the bar while the pill slides**, so a shifted pill never loses its referent. `plotW` is measured from the live `.rail` when one exists, with a viewport-derived fallback for the first paint.

### The header states the span

The sub beside the title is the day count, derived from the view:

```js
view.length < WIN ? view.length + " days" : "last " + WIN + " days"
```

It counts up — `18 days`, `24 days` — until the window stops growing, then reads `last 30 days` permanently. This is the panel's only defence against being read as a calendar month, and it costs no vertical space. An earlier attempt put the span in a caption line under the header; it worked but added 24px to an already tall panel, and the sub was the better home. **Never hard-code the string.**

### The axis

Dated ticks every 5 days, **anchored to the last day and walked backwards**:

```js
const STEP = 5, ticks = [];
for (let i = n - 1; i >= 0; i -= STEP) ticks.unshift(i);
```

Anchoring to the end rather than the start matters twice over: the as-of date always lands on a tick, and the spacing stays exactly 5 days instead of leaving a ragged gap at the newest, most-read end. The count of labels therefore grows with the window — four at 18 days, six at 30 — always evenly spaced.

Labels centre on their tick with no edge correction; the inset scale plus the panel's 22px padding keeps them clear. Interior ticks get a gridline in the plot above; the two endpoints do not, since they sit at the plot's own edges. No y-axis ticks — the marked points and the average line carry the numbers.

---

## 6b. Scrubbing the chart — reading one day

Pressing anywhere on the plot and dragging reads out that day. This is the dashboard's second interaction, and it exists because the chart necessarily marks only three days — the average, the low and the biggest — while the other 27 carry real numbers no label has room for.

### What it shows

A crosshair, a dot on the balance line, and a pill in the plot's top band carrying three things:

| Line | Content | Style |
|---|---|---|
| Day | the day's own axis label | 10px/600, `.09em` tracking, `rgba(255,244,236,.55)` |
| Balance | running balance that day | 13.5px/700, `-.02em`, accent teal |
| Spent | that day's outgoing | 10.5px/600, orange |

A day with no spending reads **"nothing spent"** rather than "0 spent". Quiet days are common and worth naming plainly; a zero reads like missing data.

**A negative balance carries a true minus sign** (`−`, U+2212). The shared `fmt()` helper is unsigned by design — every other figure on the dashboard is a magnitude — so the sign is added at the readout. Without it the pill printed `510` for a balance of `−510` while the line drew that point below the zero rule, and the graphic contradicted the number. This matters in any pre-salary stretch, where the running balance dips below zero for days at a time.

**The pill sits in the plot's top band, not at the fingertip.** On a phone the fingertip is under the thumb. It uses the same clamp as the biggest-day pill — centred on the day when it fits, otherwise flush to the edge it was crossing — so the readout never leaves the panel.

### The day under the finger

The index is found by **inverting the panel's own `X(i)`**, never by re-deriving a scale:

```js
const t = ((clientX - r.left) / r.width * 100 - pad) / (100 - 2 * pad);
const i = Math.max(0, Math.min(n - 1, Math.round(t * (n - 1))));
```

`pad` is the same half-bar inset from §6a. Rounding rather than flooring means each day owns the space nearest it, so the ends are reachable: dragging to either edge always lands on the first or last day, never one short.

The readout only updates when the index actually changes — dragging within one day's span does no work.

### What fades, and why

While a day is being read, the static annotations drop back: the marker labels and gridlines to `opacity:.16`, the biggest-day pill and its stem to `.18`. They occupy the same top band as the readout and would otherwise collide with it. On the rail, the day's own bar goes to full opacity and every other bar to `.22`, so the spend figure in the pill has a visible source.

All of it is a `.18s` opacity transition; the crosshair, dot and pill fade in over `.14s`.

**The rail's bars are restored from their own inline opacities, not reset to 1.** The rail encodes quiet days at `.18` (§6a) — resetting would erase that distinction, silently, and only on days after a scrub. Each bar's original value is captured on `pointerdown` and put back on release.

### Touch behaviour

```css
.chart { touch-action: pan-y; cursor: crosshair }
```

**`pan-y`, not `none`.** The plot is 200px tall in a scrolling page: a vertical drag that started on the chart must still scroll the page. Horizontal movement scrubs, vertical movement hands off to the browser and fires `pointercancel`, which ends the read. Setting `touch-action:none` here traps the scroll and makes the page feel stuck.

The read ends on `pointerup` **and** `pointercancel` — the second is what covers a scroll hand-off, an interrupting phone call, or a finger leaving the screen edge.

### Why it is a drag and not a tap

Tapping a tile opens something that stays open; scrubbing the chart is a transient read that ends when you let go. Nothing persists, so nothing needs dismissing. The two interactions do not overlap: the mosaic's tap opens a band, the chart's drag reads a value, and neither is available where the other lives.

---

## 7. Updating with new data

**How each rule behaves as the data grows.** Nothing here needs hand-tuning when the workbook is updated — but check each one still holds:

- **Tile type sizes** follow share of total spend, so they only change when a category's own share crosses .18 / .12 / .07. A category overtaking another does not resize either one. Verify the largest amount is still the largest type.
- **Category names** are always uppercase and fit by tightening tracking before shrinking. A longer new category name will lose tracking first; only past that does it drop below 11px.
- **More places in a category** just changes "top 5 of N". The top 5 itself is recomputed; the bars rescale to the new largest.
- **The outside-top-5 count is computed**, so it follows the data automatically — and the pill disappears entirely if every top-8 item happens to come from the top 5.
- **A place that loses its item detail** stops being tappable and loses its chevron; no code change needed.
- **New categories** enter the mosaic by rank and inherit their palette step from the 9-step ramp. Nothing about the drill-down is per-category.
- **The band's height is always measured, never assumed**, so longer place names and more items need no adjustment.
- **Scrubbing needs no adjustment either.** It inverts `X(i)` rather than assuming a day count, so it follows the window as it grows from 18 days to 30 and keeps working when each day's span narrows from ~20px to ~10px. Rounding to the nearest day is what keeps the narrower spans hittable.

**Adding new days.** Append rows to `daily`, one per calendar day including quiet ones, and update `asOf` and the workbook totals. Everything downstream recalculates: totals, left-to-spend, the split bar, both pace tiles, and the whole chart.

**As the series grows, nothing needs doing.** The 30-day window means the panel's density, average, biggest day and lowest point are all self-refreshing — the bars never thin out and no figure freezes on an old date. Keep appending to `daily` indefinitely; the hero and the mosaic go on using the full history. Do not let the chart scroll horizontally.

Everything that has to change with the window changes itself. For the record, this is what the panel does between an 18-day and a full 30-day view:

| | 18 days | 30 days |
|---|---|---|
| Header sub | `18 days` | `last 30 days` |
| Day dots | 5px | 3.5px |
| Axis labels | 4 | 6, still 5 days apart |
| Bar width | `(100/17) × .66` | `(100/29) × .66` |
| Day spacing | ~20px | ~10px |
| Low / biggest day | anywhere in the 18 | only within the last 30 |

Two consequences worth stating plainly. **The low marker and the biggest-day pill will move off old dates and never come back** — 5 Aug drops out of the window on 4 September, and the pill will name whatever the biggest day is by then. That is the design working, not data loss; the mosaic and the hero still carry the whole history. And **once past 30 days the panel's shape stops changing size** — every later update slides the window along rather than compressing it, so the layout you see at 30 days is the layout for good.

**Changing the window.** `WIN` is a single constant. 30 days suits a dashboard refreshed every 10 days: always at least two salary steps in view, and always the whole of the current month once a month has passed. Shortening it below about 14 days would leave some windows with no money-in marker at all.

**More than one money-in on the same day.** Sum them into that day's single `in` value. If two separate salaries land days apart, both get their own marker — that is correct and worth seeing.

**Category changes.** Re-rank by amount, take the top 8, recompute "Everything else". The palette is positional — rank 1 always gets `#FF5F00` — so colours reassign automatically.

**A month where spending exceeds income.** Already handled: the net goes orange with a `−` sign. If *cumulative* spending ever exceeds cumulative income, "Left to spend" goes negative — show it with a `−` and switch the hero number to `#FF5F00`.

---

## 8. Reference implementation

```
Family_Finance_Dashboard.html
```

**That file is the reference implementation. This spec and that file travel together** — neither is a complete brief alone. Do not rebuild the dashboard from this document in isolation; read the file.

This section used to hold a full copy of the source. It was removed, because a second copy of the code is a second version of the truth and it went stale twice in one day of work — describing the drill-down in §5a while its own listing still had the old accent colour, the old type-size rule and no drill-down at all. §1–7 now carry every rule as a rule. The file carries the implementation.

It also could never have done the job it was there for. The typeface is embedded as a base64 `@font-face` block of several hundred kilobytes; no listing here can include it, so this section always showed `src:url("data:font/woff2;base64,…")` with the payload elided. A rebuild from the listing alone produced a dashboard with **no embedded font** — the one failure mode §3 exists to prevent, since the file is opened offline from a chat attachment.

### What to keep in mind when editing the file

- Update the `DATA` block only. Everything else derives from it.
- Self-contained: no build step, no dependencies, **no external requests at all**.
- Keep the embedded `@font-face`. Never reintroduce a `<link>` to fonts.googleapis.com.
- Before saving, run §9.

### The two pieces prose cannot carry

Everything else in §1–7 is stated as a rule you can implement freely. These two are geometry — the exact arithmetic matters and a re-derivation will be subtly wrong.

**Chart scale and path construction.** The single x-scale shared by the line and the rail, inset half a bar width at each end, is what makes bar spacing even end to end:

```js
function buildChart() {
  const n = view.length;
  const bmax = Math.max(...view.map(p => p.bal), 0);
  const bmin = Math.min(...view.map(p => p.bal), 0);
  /* Every day sits on ONE x-scale, inset at each end by half a bar width. The
     inset is what makes the bar spacing uniform. On a flush 0-100 scale the end
     bars have to be nudged inward so they don't straddle the edge, which closes
     the gap to their neighbours — the first two and last two bars visibly join.
     Insetting instead lets every bar centre on its own day. */
  const barW = (100 / (n - 1)) * 0.66;
  const pad  = barW / 2;
  const X = i => pad + i / (n - 1) * (100 - 2 * pad);
  const Y = v => 15 + (bmax - v) / (bmax - bmin) * 72;   // bands for labels above/below
  const pts  = view.map((p, i) => X(i).toFixed(2) + "," + Y(p.bal).toFixed(2));
  const line = "M" + pts.join(" L");

  /* Dates every STEP days, anchored to the LAST day and walked backwards, so the
     as-of date always lands on a tick and the spacing stays exactly STEP. */
  const STEP = 5, ticks = [];
  for (let i = n - 1; i >= 0; i -= STEP) ticks.unshift(i);

  /* Labels sit ABOVE their dot (money-in, always a local peak) or BELOW it (the
     low, always a local trough) — never beside it. Placing them alongside the
     line put a 70-80px label across the neighbouring days, which forced the
     per-day dots to be dropped exactly on the steep runs that matter. Their own
     vertical band costs nothing and holds at any window length. */
  const label = (i, v, text, color, above) => {
    const x = X(i);
    const sx = x <= 12 ? "0" : x >= 88 ? "-100%" : "-50%";   // keep inside the plot
    return `<span class="clab" style="left:${x.toFixed(2)}%;top:${Y(v).toFixed(2)}%;color:${color};
      transform:translate(${sx},${above ? "calc(-100% - 11px)" : "11px"})">${text}</span>`;
  };
  const dot = (i, v, color) =>
    `<span class="cmark" style="left:${X(i).toFixed(2)}%;top:${Y(v).toFixed(2)}%;background:${color}"></span>`;

  const low = view[lowIdx];
  const marked = view.flatMap((p, i) => p.in > 0 ? [{ i, v: p.bal, text: "+" + fmt(p.in), color: TEAL, above: true }] : []);
  marked.push({ i: lowIdx, v: low.bal, text: "low " + fmt(low.bal), color: ORANGE, above: false });
  const marks = marked.map(m => dot(m.i, m.v, m.color) + label(m.i, m.v, m.text, m.color, m.above)).join("");

  /* Dots shrink as the window fills, so they read as separate days at 375px
     instead of beading into a thick line. */
  const dR = n <= 20 ? 5 : n <= 26 ? 4 : 3.5;
  const days = view.map((p, i) => (i === lowIdx || p.in > 0) ? "" :
    `<span class="cday" style="left:${X(i).toFixed(2)}%;top:${Y(p.bal).toFixed(2)}%;
      width:${dR}px;height:${dR}px;box-shadow:0 0 0 ${(dR / 2.5).toFixed(1)}px rgba(12,6,3,.75)"></span>`).join("");

  /* Spend rail: one bar per day on the same x-scale as the line above, so a tall
     bar sits directly under the drop it caused. Every bar is centred on its day
     — the inset above is what keeps the end ones inside the box. */
  const maxOut = Math.max(...vOut);
  const peakIdx = vOut.indexOf(maxOut);
  const bars = view.map((p, i) => {
    const pk = i === peakIdx;
    return `<i style="left:${X(i).toFixed(2)}%;width:${barW.toFixed(2)}%;transform:translateX(-50%);
      height:${Math.max(1, p.out / maxOut * 100).toFixed(1)}%;
      background:${pk ? "#FF8A3D" : ORANGE};opacity:${pk ? 1 : p.out ? .85 : .18}"></i>`;
  }).join("");
  const avgY = (1 - perDay / maxOut) * 100;
  const peak = view[peakIdx];
  /* The label is a filled pill centred over its own bar, resting on a short stem
     that lands on the bar's top edge, so there is no doubt which day it names.
     Clamped at the plot edges — the stem keeps pointing even when the pill has
     to shift inward. The rail carries 36px of margin above it so the pill always
     clears the "avg" figure in the caption row. */
  const pkX = X(peakIdx);
  const pkText = peak.label + " \u00B7 " + fmt(maxOut);
  /* The pill sizes itself to its text, so a longer date or a seven-figure amount
     widens the pill — the text can never spill out of it. What does need
     guarding is the pill running past the panel edge, so it is measured and
     shifted by the smallest amount that keeps it inside: centred when it fits,
     otherwise flush to whichever edge it was crossing. The stem stays on the
     bar regardless, so the pill can slide without losing its referent. */
  const plotW = document.querySelector(".rail")?.clientWidth
    || Math.max(269, Math.min(430, window.innerWidth) - 76);
  const pillW = textW(pkText, 10.5, 600, .04) + 20;      // 10px padding each side
  const pkCx  = pkX / 100 * plotW;
  let dx = -pillW / 2;
  if (pkCx + dx < 0) dx = -pkCx;
  if (pkCx + dx + pillW > plotW) dx = plotW - pillW - pkCx;
  const rail = `<div class="rail">
      ${bars}
      <span class="ravg" style="top:${avgY.toFixed(1)}%"></span>
      <span class="rstem" style="left:${pkX.toFixed(2)}%"></span>
      <span class="rpeak" style="left:${pkX.toFixed(2)}%;
        transform:translate(${dx.toFixed(1)}px,calc(-100% - 7px))">${peak.label} &middot; ${fmt(maxOut)}</span>
    </div>`;

  const grid = ticks.filter(i => i > 0 && i < n - 1)
    .map(i => `<span class="cgrid" style="left:${X(i).toFixed(2)}%"></span>`).join("");

  const axis = `<div class="cax">${ticks.map(i =>
    `<span class="ctick" style="left:${X(i).toFixed(2)}%"></span>` +
    `<span class="cxlab" style="left:${X(i).toFixed(2)}%">${view[i].label}</span>`
  ).join("")}</div>`;

  return `<div class="chart">
      ${grid}
      <svg viewBox="0 0 100 100" preserveAspectRatio="none" aria-hidden="true">
        <defs><linearGradient id="gfill" x1="0" y1="0" x2="0" y2="1">
          <stop offset="0" stop-color="${TEAL}" stop-opacity=".30"/>
          <stop offset="1" stop-color="${TEAL}" stop-opacity="0"/>
        </linearGradient></defs>
        <path d="${line} L${X(n-1).toFixed(2)},100 L${X(0).toFixed(2)},100 Z" fill="url(#gfill)"/>
        <path d="${line}" fill="none" stroke="${TEAL}" stroke-width="2"
          stroke-linejoin="round" stroke-linecap="round" vector-effect="non-scaling-stroke"/>
      </svg>
      <span class="czero" style="top:${Y(0).toFixed(2)}%"></span>
      <span class="clab" style="left:0;top:${Y(0).toFixed(2)}%;transform:translateY(-160%);
        color:rgba(255,244,236,.38);font-size:10px;letter-spacing:.08em">0</span>
      ${days}${marks}
    </div>
    <div class="raillab" style="margin-top:18px"><span>Spent per day</span><b>avg ${fmt(perDay)}</b></div>
    ${rail}
    ${axis}`;
}
```

**Row grouping for the mosaic.** Which categories share a row:

```js
const groups = [];
if (cats.length) groups.push([0]);
let i = 1;
while (i < cats.length && i < 7) { groups.push(cats[i+1] ? [i,i+1] : [i]); i += 2; }
while (i < cats.length) { groups.push([i]); i += 1; }
```

---

## 9. Checklist before sending it to the family

- [ ] Every figure matches the workbook exactly — no rounding, no invented numbers
- [ ] Totals came from the workbook, not from recomputing them
- [ ] `daily` covers every calendar day from the first transaction to the as-of date, quiet days included
- [ ] Chart labels stay inside the panel at 375px wide, and read clearly over the line and fill
- [ ] The low marker sits after the first money-in, not on the opening negative stretch
- [ ] Axis dates are an exact 5-day step and the last tick is the as-of date
- [ ] Rail bars align with the line above and stay inside the plot box at both ends
- [ ] The rail's average line and the "Per day" tile show the same figure
- [ ] Nothing overlaps the newest bars at the right of the rail
- [ ] Day dots read as separate dots at 375px, not as a thickened line
- [ ] **Every** day carries a dot — including the steep runs — at 375px and at a full 30-day window
- [ ] No marker label overlaps a dot, a bar, or another label at 375px
- [ ] The biggest-day callout unmistakably points at one bar
- [ ] The header sub states the real span and matches what is plotted
- [ ] Bar spacing is even end to end — no two bars touching at either edge
- [ ] The pill stays inside the panel with the peak on the first day and on the last
- [ ] No footer and no source line — the page ends after "Day by day"
- [ ] The file has **zero external references** — font embedded, nothing fetched at open
- [ ] Opens correctly with no network connection
- [ ] Mosaic areas sum to 100% of total spending ("Everything else" present if needed)
- [ ] No tile clips its name or amount at 375px wide (iPhone SE / mini)
- [ ] Glass frosts on iOS Safari — `-webkit-backdrop-filter` present
- [ ] No horizontal scrolling at any width
- [ ] Safe-area padding respected; nothing under the notch or home indicator
- [ ] Negative figures use `−` (U+2212) and go orange
- [ ] Opens correctly as a single file with no local dependencies

**The drill-down**

- [ ] Every tile is a real button and opens its own detail band, including "Everything else"
- [ ] The band opens under the tapped tile's own row, never at the end of the panel
- [ ] Tapping the open tile again closes it; opening another closes the first as one gesture
- [ ] Unopened tiles desaturate **and** darken — `saturate(.35) brightness(.62)`, not brightness alone
- [ ] The lifted tile does not overlap its neighbours (siblings shrink as it grows)
- [ ] The largest amount on the mosaic is set in the largest type, at every data update
- [ ] **The static pre-render matches `render()`** — reload and watch the mosaic: no tile changes size, colour or position after the first frame
- [ ] Group counts read "top 5 of N" only when N exceeds what is shown; singular when N is 1
- [ ] Only one place unfolds at a time
- [ ] A place with no item detail is not tappable and shows no chevron
- [ ] Place bars scale to the largest place in that category, not to the category total
- [ ] Item dates stay hidden until a tap lights that row
- [ ] Lit rows cascade rather than appearing as a block
- [ ] The two highlight modes are mutually exclusive and keep their own colours (orange / teal)
- [ ] The outside-top-5 pill appears only when the count is non-zero, and fills solid when pressed
- [ ] Pill and place rows are at least 44px tall
- [ ] Repeated toggling of any highlight or unfold leaves the band's height exact — no drift
- [ ] Rotating the phone with a band open does not replay the open animation
- [ ] The band does not clip or overflow at 375px

**Scrubbing**

- [ ] Pressing anywhere on the plot reads out a day; dragging moves the read
- [ ] Dragging to either edge reaches the first and last day
- [ ] The pill stays inside the panel at both ends
- [ ] A day with no spending reads "nothing spent", not "0 spent"
- [ ] Marker labels, gridlines and the biggest-day pill fade back while reading
- [ ] The scrubbed day's rail bar lights and the others dim
- [ ] **After release, quiet days are still dimmer than busy ones** — bars restore to their own opacity, not to 1
- [ ] A vertical drag starting on the chart still scrolls the page
- [ ] Releasing outside the chart, or scrolling away mid-drag, ends the read cleanly

---

## 10. Things that were deliberately decided

Worth knowing so they don't get "improved" away:

- **The hero answers "how much is left to spend?"** — not net worth, not this month's gap.
- **Area, not bars, for categories.** The mosaic makes Housing's dominance visible without reading a single number.
- **Amount above name in tiles.** The figure is the point; the category is the caption.
- **One accent orange plus one cool contrast.** Teal only ever means money in or money kept. No third accent.
- **Backdrop is a single ember glow**, not multiple colour blobs — the earlier multi-colour version fought with the mosaic.
- **No percentage labels on tiles.** Area already carries share; the numbers would be noise.
- **Days, not months, are the unit.** Salary lands in one lump and tracking began mid-cycle, so month-vs-month comparison misleads. A running balance is honest about both.
- **No monthly net figures anywhere.** They were the single most confusing thing in the previous version. The chart shows the same information as a shape, which needs no explaining.
- **No savings target yet.** The family started tracking in late July 2026 and has no target to measure against. The dashboard answers "is the gap widening or shrinking?" — the pace tiles and the chart's trajectory. Add a goal line only when they set a goal.
- **Two labelled points on the line, not a full axis.** The salary steps and the low point are the story; per-point value labels would bury it.
- **A dot per day plus faint 5-day gridlines.** Both asked for directly. The dots make the line countable; the gridlines make it placeable. The rail below is not a substitute — it answers "how much", not "which day".
- **Marker labels above/below their dot, never beside it.** Side-by-side labels and per-day dots compete for the same horizontal band, and the dots lose on exactly the steep runs where they earn their keep. Do not "tidy" the labels back alongside the line.
- **The panel is a rolling 30-day window, on purpose.** Asked for directly: the dashboard is updated every ~10 days and must still make sense at the end of each month, with no figure that goes stale. An all-time panel would smear the bars and freeze the average, the biggest day and the low on old dates.
- **The rail is stacked under the line, not beside it.** Sharing one x-scale is the point — a tall bar sits directly under the drop it caused.
- **The typeface is embedded, never linked.** These files are opened offline, on phones, from a chat attachment.
- **No footer and no navigation of any kind.** Removed at the user's request. The page is also shared as a lone WhatsApp attachment, so a relative link to any other file would not resolve on the recipient's phone — this page has to stand entirely on its own.
- **The biggest day is marked on the bar, not just above the rail.** Bar tint, stem and centred pill all point at one day. `#FF8A3D` is a tint of the core orange already used for links, not a new accent.
- **The callout is a filled pill, not an arrow.** An earlier version used a small triangular caret; it was the only hard-cornered shape in a design built entirely on soft radii. Chosen from four options against a rounded cap, a dot-and-stem and a soft teardrop. Any future pointer must be a rounded form.
- **The rail's average is the same number as the "Per day" tile.** The tile states it, the rail shows which days beat it. Two views of one figure, never two figures.
- **Two interactions, both direct.** Tapping a tile opens its detail; dragging across the chart reads a day. No filters, no date pickers, no tabs, no buttons that aren't the data itself. The dashboard opens calm and closed — the user's phrase, and the standard any new interaction has to meet.
- **Scrubbing is transient, the drill-down persists.** A read that ends when you lift your finger needs no dismissing; a band you opened deliberately should stay until you close it. That asymmetry is intentional.
- **The detail opens under its own row, not at the bottom of the panel.** The tile you tapped and the answer it gives stay adjacent — the earlier version opened a sheet at the far end of the mosaic and the connection was lost.
- **"Places", not "vendors".** A family does not have vendors.
- **Dates are hidden until a tap asks for them.** Chosen over showing a date on every row: eight permanent sub-lines made the band a third longer and buried the amounts.
- **Two highlight colours, and only two.** Orange means "from the place you tapped"; teal means "from outside the top 5". A third mode would need a third colour and there isn't one to spare.
- **Type on tiles is sized by amount, not by tile.** A treemap already encodes size as area; the numbers must agree with it rather than with the row-packing accident that gave them their height.
- **The unopened tiles desaturate as well as darken.** Brightness alone reads as a bug. Restored once after a port dropped it.
- **No legend on the panel.** Every element labels itself in place — the salary steps, the low point, the biggest day, the average. A legend would also have to claim orange means one thing when it marks both the rail bars and the low dot.
