# Fashion Frenzy

A single-file, offline-capable browser game about running a seamstress atelier. You open the
shop each morning, take order tickets from guests, buy fabric, cut pattern pieces by hand on a
canvas, assemble them, price the finished garment, and try to make the sale.

Everything &mdash; markup, styles, game logic, and the scene artwork &mdash; lives in one
`index.html`. There is no build step, no framework, and no server. Open the file in a browser and
it runs.

---

## Table of contents

1. [Quick start](#quick-start)
2. [File structure](#file-structure)
3. [What the game does](#what-the-game-does)
4. [Shop upgrades](#shop-upgrades)
5. [Core systems and functions](#core-systems-and-functions)
6. [Local storage keys](#local-storage-keys)
7. [Issues and things to improve](#issues-and-things-to-improve) &rarr; [todo.md](todo.md)

---

## Quick start

Open `index.html` directly in a browser, or serve the folder:

```
python3 -m http.server 8080
```

Then visit `http://localhost:8080/`.

Serving over HTTP (rather than `file://`) is required for the web manifest and the
`favicon.png` / `assets/` references to resolve correctly. The scene artwork itself is embedded
in the HTML, so the game is playable even from `file://`.

**Target platforms.** The layout is portrait-first and capped at 560&nbsp;px wide, designed for
phones and installable as a PWA on iOS and Android. It works fine on desktop as a narrow column.
All interaction uses Pointer Events, so mouse, touch, and stylus all work.

---

## File structure

```
FashionFrenzy/
  index.html               the entire game: HTML, CSS, JS, and the base64 scene art
  manifest.webmanifest     PWA metadata (name, colors, portrait orientation, icons)
  favicon.png              app icon / favicon / apple-touch-icon (1254 x 1254)
  README.md                this file
  todo.md                  known issues and improvements
  assets/
    fashionfrenzy.png      copy of the icon art
    fashionfrenzy_scene.png  full-resolution atelier scene (source for the embedded JPEG)
  .git/                    version control
  .DS_Store                macOS folder metadata (see issue 32)
```

### Inside `index.html`

The file is one document with three regions:

| Region | Approx. lines | Contents |
| --- | --- | --- |
| `<head>` + `<style>` | 1&ndash;653 | Meta tags, PWA hooks, Google Fonts link, and the complete stylesheet as CSS custom properties plus component classes |
| `<body>` | 655&ndash;663 | Five empty mount points: `#hud`, `#scene`, `#tray`, `#modal-root`, `#toast` |
| `<script>` | 664&ndash;4074 | One IIFE in strict mode containing all game code |

4,086 lines in total, of which line 669 alone is the 182&nbsp;KB base64 scene image.

The script is organised into labelled sections, in order:

```
SCENE_IMG (base64)  ->  storage keys + settings  ->  STATIC DATA  ->  UPGRADES
->  TIME CONSTANTS  ->  STATE  ->  HELPERS  ->  ORDER GENERATION  ->  SCORING
->  CUTTING  ->  CUSTOM ORDERS  ->  CATALOG & EXCLUSIVES  ->  SVG piece rendering
->  ASSEMBLY  ->  CUSTOMER DIALOGUE  ->  ORDER FLOW  ->  CLEARANCE  ->  DAY CYCLE
->  SAVE / LOAD  ->  RENDER: HUD / SCENE / TRAY / MODALS  ->  EVENTS  ->  INIT
```

### The five mount points

- **`#hud`** &mdash; top bar: cash, reputation (tap to flip between stars and a decimal), total
  sales, and a day/clock card that doubles as a progress bar filling toward closing time. Two
  buttons sit top-right: a list icon opening the [Ledger](#the-ledger), and the hamburger menu.
- **`#scene`** &mdash; the atelier artwork with invisible hotspot buttons pinned to features in the
  image (pegboard &rarr; upgrades, thread spools &rarr; fabric shop, cubby &rarr; clearance rack,
  pinned cards &rarr; catalog).
  When an order is in progress, the working panel is overlaid directly on the wooden table.
- **`#tray`** &mdash; the horizontal strip of order tickets at the bottom.
- **`#modal-root`** &mdash; every bottom-sheet and full-screen modal.
- **`#toast`** &mdash; transient notification pill.

---

## What the game does

### The day loop

Each in-game day runs from **10:00&nbsp;AM (`OPEN_MIN` = 600)** to **5:00&nbsp;PM
(`CLOSE_MIN` = 1020)**, a span of 420 in-game minutes. How much real time that takes is a player
setting (1&ndash;60 minutes, default 10). A `setInterval` fires every 200&nbsp;ms and advances the
clock by the elapsed wall-clock delta scaled through `minPerMs()`.

1. **Closed.** A veil over the scene offers *Open the Shop*, *Order Fabric First*, or
   *Shop Upgrades* (the veil covers the scene hotspots, so the pegboard needs its own way in).
2. **Open.** Guests arrive on a timer. At most **3** orders (`MAX_ORDERS`) can be in play at once,
   counting the one being worked. New arrivals stop 40 minutes before close.
3. **Summary.** At closing time the day ends (or waits for the current order to finish) and a
   summary sheet reports sales, earnings, refusals, walkouts, reputation change, and cash.

Meta screens pause the clock &mdash; menu, gameplay settings, upgrades, how-to-play, the ledger,
the day summary, and the sale result. Crafting does not, and neither do the fabric shop, clearance rack, or
catalog: the workday stays under pressure while you cut and while you shop.

### Guest arrival cadence

Two modes, chosen in Gameplay settings:

- **Reputation traffic ON (default).** Reputation (0&ndash;100, displayed as 0&ndash;5 stars) drives
  volume. Flat ~10 guests/day up to 2.5 stars, ramping linearly to a target of 70/day at 5 stars.
  `nextGuestGap()` scales the baseline 28&ndash;55 minute spread to hit that average.
- **Reputation traffic OFF.** A steady 3 guests per in-game hour regardless of reputation.

Either way, reputation scales what guests will pay via `repF = 1 + (reputation - 50) / 220`.

### Budgets

A guest's budget starts from the garment's size (`yards &times; 6.5`), multiplied by a
`1.5`&ndash;`2.6` taste roll and by `repF`. Two things can lift it above that:

- **Wealthy clientele.** Once High-End Fabrics is unlocked, `wealthRoll()` gives roughly 42% of
  arrivals a `2.4`&times;&ndash;`5.2`&times; multiplier.
- **Patrons.** Independently, and whether or not any upgrade is owned, **8%** of guests are a
  patron. Rather than a flat band, `patronBudget()` sizes their purse to the garment they actually
  came in for: `topFairPrice()` computes what the finest possible version would fairly fetch
  &mdash; every piece cut from the dearest bolt in the shop ($64/yd made up) and finished
  flawlessly &mdash; then adds `PATRON_MARGIN` (20%) on top as the shop's margin, with a
  &plusmn;15% spread between patrons. A $200 floor covers the smallest pieces, and an existing
  higher budget is never lowered.

That makes a patron's purse scale with the ask:

| Garment | Yards | Dearest version fairly fetches | Typical patron purse |
| --- | --- | --- | --- |
| Lingerie | 1 | $141 | $200 (floor) |
| T-Shirt / Pants | 2 | $282 | ~$341 |
| Coat / Dress / Outfit | 3 | $422 | ~$508 |
| **Ballgown** | 6 | **$845** | **~$1,003** |

A custom brief or an exclusive has no fixed pattern pieces, so it is valued generically at
`PATRON_BRIEF_YARDS` (4 yards) of good work.

The purse cannot be captured with cheap cloth. A purple polka-dot ballgown in cotton fairly
fetches $132, so asking a patron's $1,000 for it drives the price factor to zero and the sale
fails outright. The same gown in cashmere fetches $845 fairly, sells comfortably at $1,000, and
nets **$748 on $252 of cloth**. Patron tickets are flagged in the order detail so the player knows
to reach for the good bolts.

Ordinary budgets stay modest: the median guest is still around $35.

### Three kinds of order

| Type | Generator | What the guest specifies | Player workflow |
| --- | --- | --- | --- |
| **Standard** | `generateGuest()` | Garment type, 1&ndash;2 preferred colors (or "any"), a pattern (or "any"), and a budget | Pick fabric per pattern piece &rarr; cut each piece &rarr; assemble (if multi-piece) &rarr; price &rarr; present |
| **Custom** (~28% of spawns) | `generateCustomGuest()` | Colors and a budget only | Add as many free-form pieces as you like, cut any shape, arrange and layer them &rarr; price &rarr; present |
| **Exclusive** (22% chance when the catalog is non-empty) | `generateExclusiveGuest()` | A specific signature piece from your own catalog | Accept and jump straight to pricing &mdash; no cutting |

### The seven standard garments

Defined in `GARMENT_TYPES`, each with pattern pieces, yardage, an SVG outline, and a quality
weight:

| Garment | Pieces | Total yards |
| --- | --- | --- |
| T-Shirt | Bodice Panel | 2 |
| Pants | Leg Panel | 2 |
| Casual Outfit | Top, Bottom | 3 |
| Coat | Coat Panel | 3 |
| Lingerie | Bralette, Briefs | 1 |
| Summer Dress | Bodice, Skirt | 3 |
| Ballgown | Bodice, Grand Skirt, Sleeves | 6 |

Any garment can be switched off in the **Catalog**, and guests will stop asking for it.

### Fabric

Four materials &times; eight colors &times; three patterns = **96 bolts**, generated at load. The
base price per yard is `4 + round(rarity &times; 2) + patternSurcharge`, where rarity runs from 0.7
(ivory, charcoal) up to 1.6 (royal purple), and the surcharge is 0 for solid, +2 stripes, +3 polka
dot. That base is then scaled by the material:

| Material | Cost multiplier | Value multiplier | Available |
| --- | --- | --- | --- |
| Cotton | 1.0 | 1.0 | From day one |
| Wool | 2.1 | 2.9 | High-End Fabrics upgrade |
| Silk | 3.0 | 4.4 | High-End Fabrics upgrade |
| Cashmere | 4.2 | 6.4 | High-End Fabrics upgrade |

Every bolt therefore carries two numbers: `pricePerYard`, what it costs at the counter, and
`valuePerYard`, what a yard of it is worth once made up into a garment. They are equal for cotton
and increasingly divergent for luxury cloth, which is what makes silk and cashmere a margin play
rather than simply an expense.

The 24 cotton bolts are generated first, so the original ids `f0`&ndash;`f23` keep their exact
meaning and saves written before the upgrade existed still load correctly.

Fabric is bought in 2-yard increments from the shop, filtered by material and then grouped
color-first.

In the cutting menu, each premium swatch is stamped with its material's initial so the cloth is
identifiable at a glance without opening a tooltip: **W** wool, **S** silk, **C** cashmere. Cotton
swatches are left bare. The letter is drawn in paper cream with an ink outline (`paint-order:
stroke fill`), so it stays legible on ivory and charcoal alike, and is `pointer-events: none` so it
never intercepts a tap.

### Cutting &mdash; the core minigame

This is what the game is actually about. The chosen fabric is drawn as a 200&times;200 unit square
of cloth on a canvas, with the target pattern piece printed on it as dashed guide lines.

- The guide polygon is resampled into evenly spaced points (`polygonResample`, ~5 units apart).
- As you drag the blade, `markAlong` / `markNear` walk the path and mark every guide sample within
  `CUT_TOL` (16 units) as cut, recording the closest approach.
- When **88%** of the guide has been cut through (`CLOSE_COVERAGE`), the piece is finished.
- **Precision** = `clamp(100 - averageDeviation &times; 7, 30, 100)`.
- **Final quality** = `matchQuality &times; (0.85 + 0.15 &times; precision/100)`, where `matchQuality`
  is 60 for a color hit (15 for a miss) plus 40 for a pattern hit (10 for a miss) &mdash; so 100 is
  a perfect match, 25 a total miss.
- Critically, **the outline you actually cut becomes the piece.** A lopsided cut stays lopsided on
  the mannequin, in the catalog, and in the sale screen.

Yardage is deducted from stock only when the piece is finished. Fabric already committed to an
in-progress piece is "reserved" (`reservedYards`) so you cannot double-spend it.

Custom orders use a variant canvas (`initFreeCanvas`) with no guide lines at all: cut any closed
shape and it becomes a piece, with yardage charged by polygon area.

### Assembly

Multi-piece standard garments open an assembly table: drag each cut piece into its dashed outline;
it snaps and locks within 55 units. Custom multi-piece designs instead get **free arrangement** &mdash;
drag to move, pinch or scroll-while-holding to resize, twist or right-drag to rotate, plus a
drag-to-reorder layer strip controlling what sits in front.

### Pricing and the sale

`fairPrice() = (madeUpValue + labor) &times; (1.2 + quality/100)`, where `madeUpValue` is the sum of
each piece's `valuePerYard &times; yards` (falling back to raw material cost for anything cut before
materials existed). You set your asking price with a slider. When you present:

```
priceFactor = price > budget ? 0 : clamp(1 - max(0, price/fairPrice - 1) &times; 0.6, 0, 1)
satisfaction = quality &times; priceFactor
accepted = (satisfaction + random(-6, +6)) >= 55
```

With **Precision matters** enabled, cutting accuracy becomes a hard gate on top of that:
85%+ always satisfies, under 65% is an automatic no-sale, and the 65&ndash;85 band is judged
against a random per-customer standard.

Reputation moves by `clamp(round((satisfaction - 55) / 9), -5, 5)`, and a rejected sale can never
raise it. Rejected garments go to the **clearance rack**, where you can sell them at 55% of base
value, recycle them for half their yardage back, or donate up to 5 at a time for +4 reputation
each (a fifth of a star).

Either way, the guest says something. The result screen quotes one of ten positive lines on a
sale, or a line matched to exactly what went wrong on a refusal.

### Customer dialogue

Failures are classified into four flaw keys &mdash; `color`, `pattern`, `precision`, `price` &mdash;
and `NEGATIVE_LINES` is keyed by every one of the fifteen combinations plus a `none` set for the
guest who simply was not in the mood. That is 42 negative lines and 10 positive ones.

Lines are authored with `[[WORD]]` markers around the thing that went wrong, and `flawify()`
rewrites each marker into `<span class="flaw">`, which renders bold and in the theme's red:

```js
'It is the wrong [[COLOR]] and far too [[EXPENSIVE]] on top of that.'
```

Every line for a given combination names every flaw in that combination, so a guest refusing on
colour and price always calls out both.

### Exclusives

Any finished piece can be named and filed in the **Exclusive catalog**, either from the pricing
screen or by starting a standalone design session from the Catalog. Once a piece is in the catalog,
guests start asking for it by name and will pay `value &times; random(1.1, 1.6) &times; repF`.

### The ledger

The list button beside the hamburger opens a full-screen **Ledger** in three parts:

1. **All-time standing** &mdash; a hero net-position figure (lifetime taken minus lifetime spent)
   over a grid of stat tiles: cash, reputation, garments sold, turned down, guests served, gross
   takings, accessory sales (or bolts in stock before that upgrade), exclusives, and the best and
   worst trading days.
2. **Balance per day** &mdash; a bar chart of each closed day's net, one bar per day, on a zero
   baseline. Tap any bar to read that day's takings, outlay, and net beneath the chart.
3. **Today's receipts** &mdash; every movement of cash today, newest first, with a running
   in / out / net total. Receipts clear when the next day begins.

Bookkeeping lives in four pieces of state. `lifetime` and `history` are cumulative and saved;
`receipts` and `dayLedger` are day-scoped and wiped by `nextDay()`. Every cash movement calls
`logReceipt(dir, label, detail, amount)` &mdash; fabric orders, upgrade purchases, garment sales,
accessory sales, and clearance sales &mdash; and `endDay()` folds the day into `history` via
`recordDayInHistory()`. History is capped at `HISTORY_MAX` (120 days) so the save cannot grow
without bound, and the chart plots the most recent `CHART_DAYS` (30), saying so when it truncates.

**On the chart's colours.** The data's job is polarity over time, so the form is bars on a zero
baseline and the palette is diverging. The obvious pairing &mdash; the shop's own green for profit
against its red for loss &mdash; was measured and rejected: those two separate by only **&Delta;E 3.5
under protanopia**, meaning a red-green colourblind player cannot tell a good day from a bad one.
Ink against red separates by **&Delta;E 18.0**, is the period-correct accounting idiom (in the black
versus in the red), and clears contrast at 14.0:1 and 5.5:1 against the paper surface. Sign is
additionally carried by position across the baseline, so the chart never relies on hue alone.

### The menu

The hamburger in the HUD opens a five-button list, in order:

| Button | What it does |
| --- | --- |
| Gameplay | The five player settings below |
| Save Game | Writes the save (disabled-looking but always available) |
| Load Game | Restores the save by hand; disabled when none exists |
| New Game | Starts over from Day 1 |
| How To Play | The full guide, on its own screen |

*How To Play* is a dedicated screen (`bodyHowTo()`), not a wall of text stapled to the bottom of the
menu, and it covers the day loop, custom orders, what each scene hotspot does, both upgrades, and
where the settings live.

### Player settings

| Setting | Default | Effect |
| --- | --- | --- |
| Guest patience | off | Waiting guests eventually leave, costing reputation |
| Precision matters | off | Cutting accuracy becomes a hard gate on the sale |
| Reputation traffic | on | Reputation drives how busy the shop is |
| Selling help | on | Color-codes quality, precision, and price by sale odds |
| Day length | 10 min | Real minutes per in-game workday (1&ndash;60) |

---

## Shop upgrades

Bought from the **pegboard above the cutting table** (and from a button on the closed-day card,
since the pegboard sits behind it while the shop is shut). Upgrades are permanent: they are written
into the save file and survive every save and load, and are cleared only by starting a new game.
An affordable upgrade puts a badge on the pegboard hotspot.

### High-End Fabrics &mdash; $400

A single purchase. Unlocks the wool, silk, and cashmere versions of every bolt the shop carries
(see [Fabric](#fabric) above), adds a material filter to the fabric shop, and starts bringing
wealthier guests through the door: `wealthRoll()` gives roughly 42% of arrivals a budget
multiplier of 2.4&times; to 5.2&times;, and those tickets are flagged as a discerning client so the
player knows luxury cloth will pay for itself. On top of that, a rare 8% of guests are patrons
whose purse is sized to the finest possible version of what they asked for &mdash; up to roughly
$1,000 for a ballgown (see [Budgets](#budgets)). Those are the customers a cashmere piece is
actually cut for. Once bought, the button is grayed out and reads **Purchased**.

### Accessories &mdash; $500, then $1000, then $2000

A three-step chain. The row always offers the next step and shows what is currently owned
underneath. Once an accessory counter exists, every piece presented has a flat **30% chance** of an
extra sale, rolled independently of the garment sale &mdash; the guest may reject the dress and
still walk out with a scarf. The payout is a random integer in the tier's range, added to cash,
lifetime sales, and the day's earnings, reported on the result screen and totalled on the day
summary.

| Step | Cost | Payout per sale |
| --- | --- | --- |
| Accessories | $500 | $10 &ndash; $30 |
| Quality Accessories | $1000 | $40 &ndash; $70 |
| High-End Accessories | $2000 | $80 &ndash; $150 |

After the third step the button is grayed out and reads **Purchased**.

---

## Core systems and functions

### State

A single mutable `state` object, created by `freshState()`, holding both persistent progress and
transient per-order and UI state:

```js
{
  cash, reputation, day, guestsServed, totalSales,
  stock,          // { fabricId: yards }
  rack,           // rejected garments
  exclusives,     // player-designed signature pieces
  catalogOff,     // { garmentId: true } for switched-off offerings
  upgrades,       // { highEndFabrics: bool, accessories: 0-3 }
  lifetime,       // cumulative ledger totals (saved)
  history,        // one entry per closed day (saved, capped at 120)
  receipts,       // today's cash movements (cleared by nextDay)
  dayLedger,      // { inAmt, outAmt } for today (cleared by nextDay)
  phase,          // 'closed' | 'open' | 'summary'
  clock, tickets, nextSpawnAt, closingPending, dayStats,
  guest,          // the order being worked
  customItems, customDraft, customArrange,
  cutting,        // { partId: cutting state }
  assembly, cutGarment, price, lastResult,
  ui              // { modal, tracePart, modalId, fabricOpen, catalogTab,
                  //   donateMode, donateSel, shopColor, shopMat, repDecimal }
}
```

Three module-level counters live outside it: `ticketSeq`, `rackSeq`, `exclusiveSeq`.

### Function reference

**Settings persistence**

| Function | Purpose |
| --- | --- |
| `loadSettings()` | Reads and type-validates `fashionfrenzy_settings`; silently ignores bad data |
| `saveSettings()` | Writes settings; returns `false` if storage throws |

**Helpers**

`clamp`, `rnd`, `pick`, `clone` (JSON round-trip), `money`, `esc` (HTML escaping),
`fabricById`, `garmentTypeById`, `partOf`, `ownedYards`, `shade` / `lighten` / `darken`,
`swatchBg` (CSS gradient per pattern), `fmtClock`, `toast`, `gamePrompt` (themed replacement for
`window.prompt`), `fabricTile` (cached 16&times;16 canvas pattern), `shapeClip` (cached CSS
`clip-path`).

**Order generation and scoring**

| Function | Purpose |
| --- | --- |
| `generateGuest()` / `generateCustomGuest()` / `generateExclusiveGuest()` | Build the three ticket types |
| `wealthRoll()` | Budget multiplier for the affluent clientele High-End Fabrics attracts |
| `patronBudget(budget, garment)` | The rare 8% roll that lifts a guest into patron territory |
| `topFairPrice(garment)` | What the finest possible version of a garment would fairly fetch |
| `spawnTicket()` | Picks which type to spawn, respecting `MAX_ORDERS` and switched-off offerings |
| `partMatchQuality(fabricId)` | 0&ndash;100 color+pattern match score for one piece |
| `currentOverallQuality()` | Weighted running quality across all pieces of the current order |
| `reservedYards(fabricId, exceptPart)` | Yardage committed to other in-progress pieces |

**Cutting**

| Function | Purpose |
| --- | --- |
| `targetPolyFor(shapeKey)` | Fits and caches a pattern outline into the cloth square |
| `polygonResample(points, spacing)` | Evenly spaced guide samples used for scoring |
| `markNear` / `markAlong` | Record blade proximity to the guide |
| `piecePercentCut(st)` | Percentage of the guide cut through |
| `addCutPoint` / `simplifyPath` / `pathToClip` | Record, reduce to &le;56 points, convert to a CSS `clip-path` |
| `selectFabricForPart` | Validates unreserved yardage, then initialises cutting state |
| `finishPieceIfDone` | Closes the piece, scores precision, deducts stock |
| `initCuttingCanvas(canvas, partId)` | DPR-aware canvas setup, drawing, and pointer handling |
| `initFreeCanvas(canvas)` | Guide-free variant for custom pieces |

**Upgrades, materials, and dialogue**

| Function | Purpose |
| --- | --- |
| `buyUpgrade(key)` | Debits cash and unlocks `'fabrics'` or the next `'accessories'` step |
| `affordableUpgrades()` | How many upgrades are buyable right now, for the pegboard badge |
| `bodyUpgrades()` | The upgrades screen, including the grayed-out Purchased states |
| `materialById` / `unlockedMaterials` | Material lookup and what the shop currently stocks |
| `fabricLabel` / `fabricFullName` | Display names; premium bolts name their material |
| `fabricValuePerYard(f)` | Made-up worth per yard, as opposed to counter price |
| `materialMark(f)` | The W / S / C initial stamped over a premium swatch in the cutting menu |
| `pickerFabrics()` | Cutting-table picker: all cotton, plus premium cloth actually in stock |
| `accessoryTier()` / `accessoryRange()` | Current accessory step and its payout band |
| `rollAccessorySale()` | The independent 30% roll and its integer payout |
| `guestLine(accepted, flaws)` | Picks what the guest says, matched to the exact failure set |
| `flawify(s)` | Rewrites `[[WORD]]` markers into red bold callouts |

**Custom orders**

`startCustomItem`, `pickCustomFabric`, `resetCustomCut`, `customCutClosed` / `customCutUsable`,
`finishCustomItem` (charges yardage by `polygonArea`), `removeCustomItem`, `initCustomArrange`,
`getTF` / `tfStyle` (per-piece scale and rotation), `customZ`, `buildCustomGarment`,
`proceedCustom` / `proceedCustomAssembly`.

**Catalog and exclusives**

`toggleOffering`, `startCatalogDesign` / `endCatalogDesign` (a synthetic guest lets the custom flow
run untethered), `addExclusive` / `finalizeExclusive`, `removeExclusive`, `renameExclusive`,
`exclusiveArt`, `toggleDonate` / `confirmDonate`.

**Rendering**

| Function | Purpose |
| --- | --- |
| `piecePolySVG(fabricId, points, w, h, strokeW)` | Draws a cut piece as a real SVG polygon filled with a fabric `<pattern>` |
| `svgPatternDef(uid, fabric)` | Per-pattern SVG fill definition |
| `garmentArt` / `customArt` / `artBoxHTML` / `fitArtTransform` | Auto-scale and centre the finished piece into a target box |
| `pieceOutlineSVG` | Small outline thumbnail in the piece list |
| `renderHUD` / `renderScene` / `renderTray` / `renderModal` / `renderAll` | Full re-render of each mount point |
| `fitScene` | Keeps the aspect-locked scene stage and on-table work panel pinned to the artwork at any viewport size |
| `modalShell(title, sub, body, full, cls)` | Shared modal chrome |
| `body*()` | One builder per screen, eighteen in all: `bodyShop`, `bodyClearance`, `bodyCatalog`, `bodyTicketDetail`, `bodyCutting`, `bodyTrace`, `bodyCustom`, `bodyCustomCut`, `bodyCustomAssembly`, `bodyAssembly`, `bodyPricing`, `bodyResult`, `bodySummary`, `bodyGameplay`, `bodyMenu`, `bodyUpgrades`, `bodyHowTo`, `bodyLedger` |

**Ledger**

| Function | Purpose |
| --- | --- |
| `logReceipt(dir, label, detail, amount)` | Records one cash movement into today's receipts and the lifetime totals |
| `recordDayInHistory()` | Folds the closed day into `history` (capped at `HISTORY_MAX`) |
| `clearDayLedger()` | Wipes today's receipts and running totals when a new day starts |
| `lifetimeNet()` | Lifetime taken minus lifetime spent, the hero figure |
| `bodyLedger` / `balanceChart` / `receiptsBody` / `ledgerStat` | The three sections of the Ledger screen |
| `barPath(x, y, w, h, r)` | An SVG bar with only its data-end corners rounded, anchored to the baseline |

**Order flow and day cycle**

`acceptTicket`, `declineTicket` (&minus;1 reputation), `buildCutGarment`, `baseValue` / `fairPrice`,
`proceedFromCutting`, `proceedFromAssembly`, `presentToGuest` (the sale calculation and the
walk-away reason list), `finishOrder`, `sellRackItem` / `recycleRackItem` / `sellAllRack`,
`buyFabric`, `openStore`, `startClock` / `stopClock` / `clockPaused` / `tick`, `endDay`, `nextDay`.

**Events**

Three delegated listeners on `document` / `window`:

- `click` &mdash; a single `switch` over `data-act` attributes handling every button in the game.
- `input` &mdash; the price slider, day-length slider, and custom item name field.
- `resize` / `orientationchange` &mdash; re-fit the scene and the trace canvas.

Everything is wired through `data-act` / `data-id` / `data-part` attributes rather than
per-element listeners, except for the canvas and drag surfaces, which attach Pointer Event
handlers directly after each render.

---

## Local storage keys

The game writes exactly **two** keys. Both are JSON strings, both are wrapped in `try/catch` so
that private-browsing or blocked-storage contexts degrade to an in-memory session rather than
throwing.

### 1. `fashionfrenzy_savegame`

Written by `saveGame()`, read by `loadGame(auto)`, existence-checked by `hasSave()`. Saving is
**manual only** &mdash; via *Menu &rarr; Save Game* or the *Save Game* button on the day summary.

**Loading is automatic.** On startup the game checks for a save and resumes it without the player
touching the menu, confirming with a *Resumed &mdash; Day N* toast:

```js
loadSettings();
if (!hasSave() || !loadGame(true)) renderAll();
```

`loadGame()` returns `true` only when a save was actually applied, so a first visit, blocked
storage, or an unreadable save all fall through to a clean Day 1 with the UI fully rendered rather
than a blank screen. A corrupt save says so (*Save file is unreadable &mdash; starting a new
game*); a first visit stays quiet. *Menu &rarr; Load Game* still works for reloading the save by
hand mid-session.

| Field | Type | What is stored |
| --- | --- | --- |
| `v` | number | Save format version. Currently hardcoded to `4` on write |
| `day` | number | Current day number |
| `cash` | number | Cash on hand |
| `reputation` | number | 0&ndash;100 reputation score (displayed as 0&ndash;5 stars) |
| `guestsServed` | number | Lifetime count of completed sales |
| `totalSales` | number | Lifetime gross revenue |
| `stock` | object | Fabric inventory as `{ fabricId: yards }`, e.g. `{ f0: 3, f7: 1.5 }` |
| `rack` | array | Clearance rack. Each entry: `{ id, garmentId, garmentName, quality, value, parts: [{ fabricId, yards }] }` |
| `rackSeq` | number | Next clearance-rack id counter, so ids stay unique across sessions |
| `exclusives` | array | Player-designed signature pieces. Each entry: `{ id, name, cutGarment, customItems, customArrange, cost, quality, value }` &mdash; including every cut piece's full point array, fabric, and arrangement transform |
| `catalogOff` | object | Switched-off standard garments as `{ garmentId: true }` |
| `exclusiveSeq` | number | Next exclusive id counter |
| `upgrades` | object | Permanent shop upgrades: `{ highEndFabrics: boolean, accessories: 0-3 }`. `accessories` is how many steps have been bought &mdash; 0 none, 1 Accessories, 2 Quality, 3 High-End |
| `lifetime` | object | Cumulative ledger totals: `{ earned, spent, sold, declined, accessories, accessoryEarned }` |
| `history` | array | One entry per closed day, capped at 120: `{ d, inAmt, outAmt, net, cash, sold, rep }` &mdash; what the balance chart plots |

**Explicitly not saved:** the in-progress order (`guest`, `cutting`, `assembly`, `cutGarment`,
`customItems`, `customDraft`, `customArrange`, `price`, `lastResult`), the live day
(`phase`, `clock`, `tickets`, `nextSpawnAt`, `closingPending`, `dayStats`), the day-scoped ledger
(`receipts`, `dayLedger` &mdash; a load always lands at the start of a day, where those are empty
by definition), `ticketSeq`, and all UI state. `loadGame()` always restores you to `phase: 'closed'` at the start of the saved day.

On load, every field is defensively defaulted: a missing `cash` falls back to 150, `reputation` to
50, `rack` and `exclusives` to empty arrays, and the sequence counters to `length + 1`. A save
written before upgrades existed (`v: 2`, no `upgrades` key) loads cleanly with both upgrades
locked, and the accessory step is clamped to the range the game actually defines.

### 2. `fashionfrenzy_settings`

Written by `saveSettings()` on **every** settings change (each toggle tap and each drag of the
day-length slider), read once by `loadSettings()` at startup. These are device preferences, kept
deliberately separate from save data so they survive New Game.

| Field | Type | Default | What it controls |
| --- | --- | --- | --- |
| `patience` | boolean | `false` | Whether waiting guests eventually give up and leave |
| `precisionMatters` | boolean | `false` | Whether cutting accuracy can lose a sale outright |
| `repTraffic` | boolean | `true` | Reputation-scaled arrival volume vs. a flat 3 guests/hour |
| `sellingHelp` | boolean | `true` | Green/yellow/red sale-odds cues on the pricing screen |
| `dayMinutes` | number | `10` | Real minutes per in-game workday, clamped to 1&ndash;60 |

`loadSettings()` type-checks each field individually and ignores anything unexpected, so a
corrupted or partial settings blob falls back to defaults per-field rather than wholesale.

**No other storage is used.** There is no `sessionStorage`, no IndexedDB, no cookies, no network
requests except the Google Fonts stylesheet, and nothing is ever sent off the device.

---

## Issues and things to improve

Tracked separately in **[todo.md](todo.md)** &mdash; 40 items across save correctness,
gameplay balance, correctness bugs, performance, assets, accessibility, and
maintainability.
