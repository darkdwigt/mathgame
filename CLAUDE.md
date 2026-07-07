# Algebra Tile Math Games — Shared Conventions

This repo is a collection of single-file algebra-tile math games for South African
Grade 11–12 students, styled with a bright, child-friendly theme (comic-style font,
pastel palette, chunky "gummy" panels/buttons) and usable on both desktop and
touchscreens/tablets. Every game is built on **one shared engine**. When you add a
new game, copy the engine verbatim and change **only** the `GAME` config object
(and the page title/header). Do not fork the behaviour.

> Rule: **all tile and block behaviours must match across every game.** A student who
> learns to play one game already knows how to play the others. No game holds the
> player's hand or walks them through steps — you give them the tiles, state the goal
> in one line, and the engine detects success automatically.

## Files

- `index.html` — Landing page linking out to all games below. Not part of the shared
  engine; just a styled grid of cards (same theme/colours as the games).
- `factoring-trinomials.html` — Factoring trinomials (arrange tiles into a **rectangle**).
- `completing-the-square.html` — Completing the square (arrange tiles into a **square**).
- `prime-composite.html` — **Not part of the shared engine.** A younger-kid game (drag
  `n` unit blocks to test if they form a non-trivial rectangle, then drag the number `n`
  into a Prime/Composite column). Different tiles and interaction model (drag-to-column
  instead of drag-into-grid), so it is intentionally a standalone file — don't try to
  reconcile it with the `GAME` contract. It originated this repo's current visual theme
  (fonts/colours/panel style) and touch-friendly Pointer Events pattern, which the two
  algebra-tile games above now also share.
- `halving-towers.html` — **Not part of the shared engine.** A younger-kid game: given
  `2n` as a tower of `2n` blocks, drag blocks out of the Start well into Tower A / Tower B
  until they're equal, then type the number sentence `n + n = 2n` (`2n` is pre-filled/given,
  only the two `n` blanks are typed) and press Check. Runs `2n = 2,4,6,8,10,12`. Blocks are
  dragged between three flexbox "well" zones (drop detected by point-in-rect, not a pixel
  grid) rather than the tile-grid engine — same visual theme and Pointer Events pattern as
  `prime-composite.html`, but its own simpler drag model suited to stacking identical blocks.

## The shared engine (identical in every game)

### Constants
- `X = 3` — an "x" length renders as **3 cells**.
- `GRID = 10` — workspace is a 10×10 cell grid.
- `CELL = 44` — the *starting* pixel size of one cell (see Adaptive fit below; it's a floor for
  small screens, not a fixed value — the board grows well past this on spacious desktops/tablets).

### Tiles (same colours, sizes, labels everywhere)
- **x²-tile**: teal (`--x2`), 3×3 cells, label `x²`.
- **x-tile**: green (`--x`), 3×1 (horizontal) or 1×3 (vertical), label `x`. Default horizontal in the tray.
- **unit-tile**: orange (`--unit`), 1×1, label `1`.

### Interactions (identical, non-negotiable)
- **Drag** with Pointer Events (`pointerdown`/`pointermove`/`pointerup`/`pointercancel`),
  which unify mouse, touch, and pen input so every game works on tablets/touchscreens as
  well as desktop (tiles have `touch-action:none`). The tile lifts out (position:fixed)
  and follows the pointer.
- **Snap to grid** on drop: nearest cell to the tile's top-left, clamped inside the grid.
- **Collision**: one tile per cell. An invalid drop (overlap, or released outside the
  workspace) returns the tile to where it came from (its previous placement, else the tray).
- **Double-tap (or double-click) an x-tile to rotate** it 90° (works in the tray and in
  place; if the rotated tile no longer fits, it returns to the tray). Detected manually by
  timing consecutive taps/clicks on the same tile (<350ms apart) rather than relying on the
  browser's native `dblclick`, so it behaves identically on touch and desktop. This is the
  ONLY rotate gesture — no single-tap rotate, no buttons.
- **Placed tiles are re-draggable** — pick any tile back up at any time before the win.
- Validation runs **after every drop and every rotate**.

### Layout (identical structure)
- Header: title + `Question N/10` + `Score`.
- Left panel: the expression, a result/identity area, controls (**Next**, **Reset**,
  **Show hint**), and a colour legend.
- Centre: the grid workspace + a single one-line goal message underneath.
- Right panel: the tile supply tray.

### Controls (identical)
- **Next** — disabled until the question is solved; advances, or shows the end card after Q10.
- **Reset workspace** — returns all tiles to the tray.
- **Show hint** — briefly flashes the top-left 3×3 corner (where the x²-tile belongs).
  Hints highlight, they do not instruct.

### Success detection (the only part that varies — via `GAME.check`)
The engine, after every change, checks: **all tiles placed** → occupied cells form a
**single solid filled rectangle** (bounding box fully covered, no gaps). It then calls
`GAME.check(Q, width, height)`. If that returns a result object, the engine:
- highlights the top row + left column and draws dimension brackets,
- shows the result identity and a celebration,
- increments the score and enables **Next**.

### Tile supply is always exact
Each question's tray holds exactly the tiles needed for the solution (same as the student
counting out tiles in class). The challenge is the **arrangement**, never guessing counts.

### Adaptive fit: grows AND shrinks, never assume a fixed cell size
`fitLayout()` starts from the breakpoint's CSS `--cell`/`--bcell`, then:
1. **Grows** the cell size (up to a per-game `CELL_MAX`/`BCELL_MAX` cap) while the page still
   fits the viewport, so the board isn't stranded tiny in the corner of a big desktop/tablet
   screen — a real bug we found and fixed (previously the cell size only ever shrank, so a
   1440px-wide window rendered the same cramped 440px board as a small laptop).
2. **Shrinks** below the breakpoint default as far as needed for a tile-heavy question (some
   trinomials/questions need far more tiles than others) so the page never needs to scroll.

**Gotcha:** always call `document.documentElement.style.removeProperty('--cell')` (or
`--bcell`) at the *start* of `fitLayout()`, before reading `cssNum(...)`. Without this, a
shrink applied on a tile-heavy question persists as an inline style override and compounds on
the *next* question too, even if that question has far fewer tiles and would otherwise fit at
full size. This bit us once already — keep the reset as the first line of `fitLayout()`.

`halving-towers.html` has no JS-driven cell size (fixed-px wells/blocks), so its equivalent of
"grow on spacious screens" is a plain `@media (min-width:1300px)` breakpoint bumping the
fixed px values instead — same intent, different mechanism because that file isn't part of
the shared engine.

## The `GAME` config contract (the per-game differences)
```
const GAME = {
  goal,                       // one-line goal string shown under the workspace
  buildSession(),             // -> array of question params (10 questions)
  exprHTML(Q),                // -> the displayed expression for question Q
  supply(Q),                  // -> [{type:'x2'|'x'|'unit', n}], EXACT tile counts
  check(Q, w, h)              // -> {identity, topLabel, leftLabel, cheer} on win, else null
};
```

- **Factoring** (`factoring-trinomials.html`): trinomials `x² + bx + c` from factor pairs `(p,q)`, `1≤p≤q≤4`.
  Win when the filled rectangle's sides give `f1=w-3, f2=h-3` with `f1+f2=b, f1·f2=c`.
  Identity `(x + f1)(x + f2)`.
- **Completing the square** (`completing-the-square.html`): `x² + bx + ?` for even
  `b ∈ {2,4,6,8,10}`. Supply: 1 x², `b` x-tiles, `(b/2)²` units. Win when the filled shape is
  a **square** (`w==h`) with side `x + b/2`. Identity `x² + bx + (b/2)² = (x + b/2)²`
  (Unicode superscripts, never `x^2`).

## Accessibility (shared across every game)
- Every page has a `@media (prefers-reduced-motion:reduce)` rule collapsing the pop/shake/
  reaction animations, and a `:focus-visible` ring (`#4d8fff`) on links/buttons/inputs so
  keyboard focus is always visible.
- Status/result regions (`#result`, `#goal`/`.hint-msg`, `#msg`) carry `aria-live="polite"`
  so screen readers announce outcomes without the user needing to hunt for them.
- **Known gap:** tile/block/chip dragging has no keyboard-operable equivalent — it's pure
  Pointer Events. Full keyboard drag-and-drop for a physical tile-arrangement game is a
  meaningfully bigger feature (needs a "pick up / move / drop" keyboard mode per game) and
  hasn't been built yet. If a future session tackles it, design it once in the shared engine
  and port it to `prime-composite.html`/`halving-towers.html` deliberately, not silently.

## Reaction images (`assets/reactions/`)
Kept small on purpose: the overlay shows them at `max-width:66vw;max-height:66vh` for ~2s, so
there's no reason to ship camera-resolution originals. Convention: resize to a **900px max
edge**, JPEG quality ~75–78 (`sips -s formatOptions 78 in.jpg --resampleHeightWidthMax 900
--out out.jpg` on macOS). Each game also preloads every reaction image at boot
(`[...HAPPY_IMAGES,...SAD_IMAGES].forEach(src=>{ const img=new Image(); img.src=src; })`) so
the *first* celebration/fail overlay pops instantly instead of waiting on a cold fetch.

## Adding a new game
1. Copy an existing file. Keep all CSS and all engine functions byte-for-byte.
2. Replace the `<title>`/header text and the `GAME` object.
3. Verify headlessly. There's no system Chromium preinstalled (`/opt/pw-browsers` doesn't
   exist on this machine) — set one up in a scratch dir: `npm init -y && npm install
   playwright && npx playwright install chromium`, then drive the page with Playwright and
   call the engine's own globals directly from `page.evaluate()` (e.g. `placeTile`,
   `doRotate`, `checkWin`) rather than simulating real pointer drags — the shared engine's
   top-level `function` declarations are already on `window` since these are classic
   (non-module) scripts. Confirm a correct arrangement triggers the win and the identity/
   labels are right for every question variant before trusting a change.

## Dev tooling: agent-skills plugin

[addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) is cloned locally
into `.agent-skills/` (gitignored, not part of this repo). It adds slash commands
(`/spec`, `/plan`, `/build`, `/test`, `/review`, `/webperf`, `/code-simplify`, `/ship`)
and skills that auto-activate during development. To use it, launch Claude Code from
this directory with:

```
claude --plugin-dir "$(pwd)/.agent-skills"
```
