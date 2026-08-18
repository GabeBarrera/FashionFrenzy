# Fashion Frenzy &mdash; todo

Known issues, inefficiencies, and things worth improving in `index.html`.
Ordered roughly by impact. See [README.md](README.md) for how the game actually works.

Every item below was re-verified against the current `index.html`; each one
describes a defect that is still present unless it is struck through and marked
*Fixed*.

---


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
   from inside the game. **Now that startup autoloads**, this bites harder: start a New Game,
   play without saving, refresh, and the *old* save comes back instead of the run in progress.
   The fix is not to delete the save silently &mdash; it is to autosave the new run (see item 1),
   or to offer an explicit *Delete Save* in the menu.

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

8. **Standard garments have no labor value.** `baseValue()` is `value + (labor || 0)`, and `labor`
   is only ever set by `buildCustomGarment()`. A hand-cut ballgown is therefore priced purely off
   the cloth that went into it, while a two-scrap custom design earns $30 of labor on top. Give
   standard garments a labor term scaled by piece count and yardage.

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

38. **4,086 lines in one file, of which one line is 182&nbsp;KB.** The single-file constraint is a
    real virtue for a game meant to be dropped on a phone and played offline, so this is not a
    call to add a bundler. But moving the scene image to `assets/` alone would take the file from
    370&nbsp;KB to about 183&nbsp;KB and make diffs readable again.

39. **No tests and no error boundary.** An exception thrown inside a `render*` function leaves the
    UI half-drawn with no recovery path. Wrapping `renderAll()` in a `try/catch` that surfaces a
    toast, plus a handful of unit tests around the scoring maths (`partMatchQuality`,
    `fairPrice`, `presentToGuest`), would catch the balance regressions that are easiest to
    introduce.

40. **`clockPaused()` is inconsistent about what counts as a meta screen.** The menu, gameplay
    settings, upgrades, how-to-play, summary, and result screens pause the clock; the fabric shop,
    clearance rack, and catalog do not. Burning in-game time while browsing the shop may well be
    intentional, but the split is not documented anywhere in the UI and reads as a bug the first
    time a day ends while you are picking fabric.
