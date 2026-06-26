# Algebra Tile Math Games — Shared Conventions

This repo is a collection of single-file algebra-tile math games for South African
Grade 11–12 students (desktop, ~1280px). Every game is built on **one shared engine**.
When you add a new game, copy the engine verbatim and change **only** the `GAME`
config object (and the page title/header). Do not fork the behaviour.

> Rule: **all tile and block behaviours must match across every game.** A student who
> learns to play one game already knows how to play the others. No game holds the
> player's hand or walks them through steps — you give them the tiles, state the goal
> in one line, and the engine detects success automatically.

## Files

- `index.html` — Factoring trinomials (arrange tiles into a **rectangle**).
- `completing-the-square.html` — Completing the square (arrange tiles into a **square**).

## The shared engine (identical in every game)

### Constants
- `X = 3` — an "x" length renders as **3 cells**.
- `GRID = 10` — workspace is a 10×10 cell grid.
- `CELL = 44` — pixel size of one cell.

### Tiles (same colours, sizes, labels everywhere)
- **x²-tile**: teal (`--x2`), 3×3 cells, label `x²`.
- **x-tile**: green (`--x`), 3×1 (horizontal) or 1×3 (vertical), label `x`. Default horizontal in the tray.
- **unit-tile**: orange (`--unit`), 1×1, label `1`.

### Interactions (identical, non-negotiable)
- **Drag** with raw mouse events (`mousedown`/`mousemove`/`mouseup`). The tile lifts out
  (position:fixed) and follows the cursor.
- **Snap to grid** on drop: nearest cell to the tile's top-left, clamped inside the grid.
- **Collision**: one tile per cell. An invalid drop (overlap, or released outside the
  workspace) returns the tile to where it came from (its previous placement, else the tray).
- **Double-click an x-tile to rotate** it 90° (works in the tray and in place; if the
  rotated tile no longer fits, it returns to the tray). This is the ONLY rotate gesture —
  no single-click rotate, no buttons.
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

- **Factoring** (`index.html`): trinomials `x² + bx + c` from factor pairs `(p,q)`, `1≤p≤q≤4`.
  Win when the filled rectangle's sides give `f1=w-3, f2=h-3` with `f1+f2=b, f1·f2=c`.
  Identity `(x + f1)(x + f2)`.
- **Completing the square** (`completing-the-square.html`): `x² + bx + ?` for even
  `b ∈ {2,4,6,8,10}`. Supply: 1 x², `b` x-tiles, `(b/2)²` units. Win when the filled shape is
  a **square** (`w==h`) with side `x + b/2`. Identity `x² + bx + (b/2)² = (x + b/2)²`
  (Unicode superscripts, never `x^2`).

## Adding a new game
1. Copy an existing file. Keep all CSS and all engine functions byte-for-byte.
2. Replace the `<title>`/header text and the `GAME` object.
3. Verify headlessly (Chromium at `/opt/pw-browsers/chromium`) that a correct arrangement
   triggers the win and that the identity/labels are right for every question variant.
