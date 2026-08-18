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
7. [Issues, inefficiencies, and things to improve](#issues-inefficiencies-and-things-to-improve)

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
  assets/
    fashionfrenzy.png      copy of the icon art
    fashionfrenzy_scene.png  full-resolution atelier scene (source for the embedded JPEG)
  .git/                    version control
```

### Inside `index.html`

The file is one document with three regions:

| Region | Approx. lines | Contents |
| --- | --- | --- |
| `<head>` + `<style>` | 1&ndash;~640 | Meta tags, PWA hooks, Google Fonts link, and the complete stylesheet as CSS custom properties plus component classes |
| `<body>` | ~642&ndash;650 | Five empty mount points: `#hud`, `#scene`, `#tray`, `#modal-root`, `#toast` |
| `<script>` | ~651&ndash;3700 | One IIFE in strict mode containing all game code |

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
  sales, and a day/clock card that doubles as a progress bar filling toward closing time.
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

1. **Closed.** A veil over the scene offers *Open the Shop* or *Order Fabric First*.
2. **Open.** Guests arrive on a timer. At most **3** orders (`MAX_ORDERS`) can be in play at once,
   counting the one being worked. New arrivals stop 40 minutes before close.
3. **Summary.** At closing time the day ends (or waits for the current order to finish) and a
   summary sheet reports sales, earnings, refusals, walkouts, reputation change, and cash.

Meta screens (menu, summary, result) pause the clock. Crafting does not &mdash; the workday stays
under pressure while you cut.

### Guest arrival cadence

Two modes, chosen in Gameplay settings:

- **Reputation traffic ON (default).** Reputation (0&ndash;100, displayed as 0&ndash;5 stars) drives
  volume. Flat ~10 guests/day up to 2.5 stars, ramping linearly to a target of 70/day at 5 stars.
  `nextGuestGap()` scales the baseline 28&ndash;55 minute spread to hit that average.
- **Reputation traffic OFF.** A steady 3 guests per in-game hour regardless of reputation.

Either way, reputation scales what guests will pay via `repF = 1 + (reputation - 50) / 220`.

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
player knows luxury cloth will pay for itself. Once bought, the button is grayed out and reads
**Purchased**.

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
  phase,          // 'closed' | 'open' | 'summary'
  clock, tickets, nextSpawnAt, closingPending, dayStats,
  guest,          // the order being worked
  customItems, customDraft, customArrange,
  cutting,        // { partId: cutting state }
  assembly, cutGarment, price, lastResult,
  ui              // { modal, tracePart, modalId, fabricOpen, catalogTab, ... }
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
| `body*()` | One builder per screen: `bodyShop`, `bodyClearance`, `bodyCatalog`, `bodyTicketDetail`, `bodyCutting`, `bodyTrace`, `bodyCustom`, `bodyCustomCut`, `bodyCustomAssembly`, `bodyAssembly`, `bodyPricing`, `bodyResult`, `bodySummary`, `bodyGameplay`, `bodyMenu` |

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

Written by `saveGame()`, read by `loadGame()`, existence-checked by `hasSave()`. Saving is
**manual only** &mdash; via *Menu &rarr; Save Game* or the *Save Game* button on the day summary.

| Field | Type | What is stored |
| --- | --- | --- |
| `v` | number | Save format version. Currently hardcoded to `3` on write |
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

**Explicitly not saved:** the in-progress order (`guest`, `cutting`, `assembly`, `cutGarment`,
`customItems`, `customDraft`, `customArrange`, `price`, `lastResult`), the live day
(`phase`, `clock`, `tickets`, `nextSpawnAt`, `closingPending`, `dayStats`), `ticketSeq`, and all
UI state. `loadGame()` always restores you to `phase: 'closed'` at the start of the saved day.

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

## Issues, inefficiencies, and things to improve

Ordered roughly by impact.

### Data loss and save correctness

1. **There is no autosave.** Progress is written only when the player explicitly taps *Save Game*.
   A refresh, a crash, an iOS tab eviction, or simply closing the PWA loses everything since the
   last manual save. Autosaving on `endDay()`, after each completed sale, and on
   `visibilitychange` would cost almost nothing.

2. **The save cannot resume a day in progress.** `saveGame()` omits `phase`, `clock`, `tickets`,
   `dayStats`, and the entire in-progress order. Saving mid-order and reloading silently discards
   the garment you were cutting and drops you at the start of the day, with the fabric already
   deducted from stock. Either persist the working state or disable the save button while an
   order is open.

3. **The `v` field is written but never read.** `saveGame()` stamps `v: 3` and `loadGame()`
   ignores it entirely. Every field is defensively defaulted, so older saves do load correctly
   today, but there is still no explicit migration path &mdash; a future format change will load
   garbage into a fresh state rather than upgrading or refusing. Add a version switch on load.

4. **`ticketSeq` is neither saved nor reset on load.** After `loadGame()` it keeps counting from
   wherever the current session left off. Harmless today because ticket ids are session-scoped,
   but it is inconsistent with `rackSeq` and `exclusiveSeq`, which are both persisted.

5. **Nothing ever calls `removeItem`.** *New Game* resets memory but leaves the old save on disk,
   so `hasSave()` still reports `true` and *Load Game* stays enabled &mdash; a player who starts over
   and then taps Load silently resurrects the abandoned run. There is also no way to clear a save
   from inside the game.

6. **Exclusives are stored uncompressed and unbounded.** Each catalog entry keeps a full
   `cutGarment` plus `customItems` plus `customArrange`, and every cut piece carries up to 56
   `{x, y}` float pairs written at full precision. The clearance rack is likewise never capped.
   A long run can push the save toward the ~5&nbsp;MB `localStorage` quota, at which point
   `setItem` throws and the player sees only "Could not save". Round coordinates to one decimal,
   cap the rack, and warn when the serialized save exceeds a threshold.

### Gameplay and balance

7. **Exclusive orders consume no fabric.** `acceptTicket()` clones the saved `cutGarment` and jumps
   straight to pricing without touching `state.stock`. Every repeat sale of a signature piece is
   pure profit with zero material cost &mdash; the strongest money exploit in the game. Deduct the
   piece's yardage (and refuse the order when stock is short).

8. **Standard garments have no labor value.** `baseValue()` is `cost + (labor || 0)`, and `labor`
   is only ever set by `buildCustomGarment()`. A hand-cut ballgown is therefore priced purely off
   its fabric cost, while a two-scrap custom design earns $30 of labor. Give standard garments a
   labor term scaled by piece count and yardage.

9. **The reputation-traffic ceiling is unreachable.** `guestsPerDayTarget()` ramps to 70 guests per
   day at five stars, but `MAX_ORDERS` is 3 and only one order can be worked at a time. Past a
   modest reputation the extra arrivals simply cannot be served, so the whole upper half of the
   traffic curve is inert. Either raise the concurrent-order cap as reputation grows, or re-scale
   the target to something a single tailor can actually serve.

10. **Spawn timing stalls against a full tray.** `nextSpawnAt` only advances when a spawn actually
    happens, so while the tray is full the condition stays true and re-evaluates every 200&nbsp;ms
    tick; the instant a slot frees, a guest materialises with no gap. Reschedule `nextSpawnAt`
    even when the spawn is suppressed.

11. **A backgrounded tab fast-forwards the clock.** `tick()` advances by `Date.now()` delta with no
    ceiling. Leaving the tab for ten minutes and returning jumps the in-game clock straight past
    closing time, ending the day and skipping every guest who would have arrived. Clamp `dt` per
    tick and pause on `document.hidden`.

12. **Comment and code disagree on the arrival cutoff.** The comment says "new orders arrive until
    an hour before close"; the code uses `CLOSE_MIN - 40`. `SPAWN_WINDOW` uses the same 40. Pick
    one and make the comment match.

13. **`dayMinutes` has two different defaults.** The `settings` object initialises it to `10`, but
    the slider handler falls back to `5` on an unparseable value (`parseInt(...) || 5`). Use one
    constant.

### Correctness bugs

14. **HTML entities leak into `textContent`.** `refreshFoot()` in `initFreeCanvas` assigns
    `'shape closed &mdash; ready to keep'` via `textContent`, which renders the literal characters
    `&mdash;` on screen instead of an em dash. Use the character (or set `innerHTML`) in that one
    place.

15. **`toast()` renders unescaped user input.** The toast body is assigned with `innerHTML`, and
    several call sites still interpolate player-supplied names without `esc()` &mdash; notably
    `finishCustomItem()`'s `toast(name + ' cut ...')`, where `name` comes straight from the item
    name field. It is only self-XSS, but a piece named `<img onerror=...>` will execute. Escape at
    the call sites, or make `toast()` escape and take an explicit opt-in for markup.

16. **Dead code path for a `plaid` pattern.** `swatchBg()`, `fabricTile()`, and `svgPatternDef()`
    all branch on `'plaid'`, but `PATTERNS` only defines solid, stripes, and dots. Either ship
    plaid or delete the three branches.

17. **A no-op line in `initLayerDrag`.** `el.releasePointerCapture && el.pointerId;` evaluates two
    expressions and discards both &mdash; clearly meant to be
    `el.releasePointerCapture(pointerId)`. The captured pointer is never released, which can leave
    the layer strip in a stuck state on some browsers.

18. **Precision is shown in green when it cannot matter.** `precisionColor()` returns
    `var(--good)` unconditionally while *Precision matters* is off, so the pricing screen reports
    a 32% cut as green. Grey it out or hide the cue instead of miscolouring it.

19. ~~**UTF-8 BOM at the top of `index.html`.**~~ *Fixed.* The file is now pure ASCII end to end.
    The codebase consistently uses HTML entities such as `&mdash;` and `&middot;` rather than
    literal characters, which is the right call for cross-platform rendering; the stray BOM before
    `<!DOCTYPE html>` has been stripped.

### Performance

20. **A 182&nbsp;KB base64 data URL is re-injected on nearly every render.** `SCENE_IMG` is set as
    the `<img src>` in `renderScene()` *and* inlined again as
    `style="background-image:url(<182KB>)"` in both `bodyTrace()` and `bodyCustomCut()`. Every
    render of those screens builds a multi-hundred-kilobyte HTML string. Reference the file in
    `assets/` (or a CSS class assigned once) instead of interpolating the data URL into markup.

21. **`renderAll()` destroys and rebuilds the entire DOM.** Almost every player action calls it,
    replacing `innerHTML` on all four mount points. That drops scroll positions in long lists,
    blurs the custom item name input mid-typing, forces the browser to re-parse the scene image,
    and re-attaches every canvas and drag listener. Most actions only need one region &mdash;
    prefer targeted `renderModal()` / `renderTray()` calls, and move toward keyed updates for the
    lists.

22. **`renderModal()` runs mid-interaction.** `finishPieceIfDone()` triggers a full modal re-render,
    which tears down and rebuilds the very canvas the player just finished dragging on. It works,
    but it is a re-entrancy hazard and a visible hitch.

23. **`svgSeq` grows without bound.** Every call to `piecePolySVG()` mints a fresh
    `<pattern>` id and a fresh `<defs>` block, so a screen showing eight catalog thumbnails emits
    eight identical gradient definitions per render. Cache one definition per fabric id.

24. **Linear scans inside render loops.** `fabricById`, `garmentTypeById`, `partOf`,
    `customItemById`, and `exclusiveById` are all `for` loops over arrays, called repeatedly from
    inside `map()` bodies. Build lookup maps once at startup.

25. **`markNear` rescans every guide sample per blade step.** `markAlong` interpolates up to 24
    sub-steps, each scanning all ~150 resample points &mdash; roughly 3,600 distance computations
    per `pointermove`. It holds up on modern hardware but is the obvious hot spot; a simple spatial
    grid or an early bounding-box reject would cut it by an order of magnitude.

26. **`clone()` is a JSON round-trip.** Used on exclusives, which carry the largest structures in
    the game. `structuredClone` is available everywhere this game runs and is substantially faster.

27. **Duplicated canvas plumbing.** `initCuttingCanvas` and `initFreeCanvas` share roughly 60% of
    their code (DPR sizing, `toLocal`, polygon tracing, pointer wiring). Extract the common
    scaffolding.

### Assets and packaging

28. **The 1.9&nbsp;MB `assets/fashionfrenzy_scene.png` is never loaded at runtime** &mdash; the scene
    is embedded as base64 JPEG in the HTML. Keep it as the design source if you like, but say so,
    or move it out of the shipped folder.

29. **`favicon.png` and `assets/fashionfrenzy.png` are byte-identical duplicates** (941&nbsp;KB
    each), and the manifest declares a single 1254&times;1254 icon for both `any` and `maskable`
    purposes. Ship properly sized 192&times;192 and 512&times;512 icons and drop the duplicate.

30. **The PWA has no service worker.** `manifest.webmanifest` makes the game installable, but with
    no offline cache the Google Fonts stylesheet fails without a network and the app falls back to
    system fonts. Since the game is already a single file, a ten-line service worker caching
    `index.html` plus the icon would make it fully offline-capable.

31. **Google Fonts is the only network dependency.** Three families are pulled from a CDN. Consider
    self-hosting or subsetting them &mdash; and note the `preconnect` points at
    `fonts.googleapis.com` but not `fonts.gstatic.com`, where the font files actually live, so the
    hint does about half of what it could.

32. **No `.gitignore`, and `.DS_Store` is present in the working tree.** Add one.

### Accessibility

33. **Pinch-zoom is disabled globally** (`maximum-scale=1.0, user-scalable=no`). This is
    understandable given that the arrangement screen uses pinch gestures, but it locks out zoom
    everywhere else in the app, which is a real barrier for low-vision players. Scope the gesture
    handling instead of disabling zoom document-wide.

34. **Modals do not trap or restore focus.** `modalShell()` produces no `role="dialog"`, no
    `aria-modal`, no initial focus, no Escape handler, and no focus return on close. Only
    `gamePrompt()` gets this right &mdash; it is a good template for the rest.

35. **The toast is invisible to screen readers.** Every confirmation, error, and precision score is
    delivered through it. Add `role="status"` and `aria-live="polite"`.

36. **Range inputs have no labels.** The price and day-length sliders carry no `aria-label` or
    associated `<label>`, so they announce only as "slider".

37. **The cutting minigame has no non-pointer path.** Cutting requires a sustained drag, which
    excludes keyboard and switch users entirely. A "cut automatically at N% precision" fallback
    would open the game up considerably.

### Maintainability

38. **3,545 lines in one file, of which one line is 182&nbsp;KB.** The single-file constraint is a
    real virtue for a game meant to be dropped on a phone and played offline, so this is not a
    call to add a bundler. But moving the scene image to `assets/` alone would take the file from
    344&nbsp;KB to about 162&nbsp;KB and make diffs readable again.

39. **No tests and no error boundary.** An exception thrown inside a `render*` function leaves the
    UI half-drawn with no recovery path. Wrapping `renderAll()` in a `try/catch` that surfaces a
    toast, plus a handful of unit tests around the scoring maths (`partMatchQuality`,
    `fairPrice`, `presentToGuest`), would catch the balance regressions that are easiest to
    introduce.

40. **`clockPaused()` is inconsistent about what counts as a meta screen.** The menu, summary, and
    result screens pause the clock; the fabric shop, clearance rack, and catalog do not. Browsing
    the shop mid-day burning in-game time may well be intentional, but it is not documented
    anywhere in the UI and reads as a bug the first time a day ends while you are picking fabric.
