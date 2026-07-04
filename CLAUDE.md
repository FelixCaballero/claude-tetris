# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Vanilla-JavaScript Tetris built on HTML5 Canvas. No frameworks, no dependencies, no build step, no `package.json`, no test suite, no linter. The entire project is three files: `index.html`, `style.css`, `game.js`.

## Running / developing

There is no build or install step. Either:

```bash
start index.html       # Windows: open directly in the default browser
```

or serve it statically (recommended, avoids any local-file/CORS quirks):

```bash
python3 -m http.server 8000
npx serve .
```

Then visit `http://localhost:8000`. Edits to `game.js` / `style.css` / `index.html` take effect on browser refresh — nothing to compile.

## Architecture

All game logic lives in `game.js` (~300 lines): module-scoped mutable `let` globals (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, etc.) plus plain functions — no classes, no modules. `index.html` only provides the DOM (a `300x600` `<canvas id="board">`, a `120x120` `<canvas id="next-canvas">` for the preview, and the HUD/overlay elements game.js writes into). `style.css` is purely presentational (dark/retro theme).

**Board model**: `board` is a `ROWS × COLS` matrix of ints. `0` = empty cell; `1–7` = a piece/color index. `COLORS` and `PIECES` are both arrays indexed `1–7` with a leading `null` at index `0`, so a cell's value indexes directly into either array.

**Piece lifecycle**:
- `randomPiece()` creates a piece object `{ type, shape, x, y }` spawned at the top-center.
- `next` holds the upcoming piece; `spawn()` promotes `next` → `current` and generates a new `next`. If the newly spawned piece immediately collides, `endGame()` fires.
- `collide(shape, ox, oy)` is the single source of truth for both wall/floor bounds and overlap with locked blocks — reused everywhere (movement, rotation, ghost projection, spawn check).
- `tryRotate()` rotates via `rotateCW()` (transpose + reverse rows) then attempts wall kicks by testing x-offsets `[0, -1, 1, -2, 2]` in order, keeping the first that doesn't collide.
- `lockPiece()` = `merge()` (bakes the piece into `board`) → `clearLines()` → `spawn()`.
- `clearLines()` scans bottom-to-top, splices out full rows, unshifts empty rows at the top, and re-checks the same row index after a splice (`r++` before the loop's `r--`).

**Game loop**: `loop(ts)` runs via `requestAnimationFrame`, accumulates elapsed time in `dropAccum`, and drops the piece one row (or calls `lockPiece()`) once `dropAccum >= dropInterval`. It calls `draw()` every frame regardless. `togglePause()` cancels/restarts the `requestAnimationFrame` chain rather than gating logic inside `loop`.

**Rendering**: `draw()` redraws everything from scratch each frame — grid lines, locked board cells, the ghost piece (via `ghostY()`, which projects `current` straight down until it would collide, drawn at `globalAlpha = 0.2`), then the current piece on top. `drawNext()` only re-renders the preview canvas when a new piece spawns (not every frame).

**Scoring/level**: line clears score via `LINE_SCORES[cleared] * level`; hard drop adds `2` points per cell dropped, soft drop `1` point per row. Level increments every 10 lines (`Math.floor(lines / 10) + 1`), which recalculates `dropInterval = max(100, 1000 - (level-1)*90)`.

**HUD/overlay**: `updateHUD()` pushes `score`/`lines`/`level` to the DOM after any state change. The single `#overlay` element is reused for both "GAME OVER" and "PAUSA" by swapping its title/score text.

## Key coupling to watch for

`<canvas id="board">`'s `width`/`height` attributes in `index.html` are literal pixel values that must stay in sync with `COLS * BLOCK` and `ROWS * BLOCK` from `game.js` — they are not computed from JS. If you change `COLS`, `ROWS`, or `BLOCK`, update the canvas attributes in `index.html` to match.

Tunable constants live at the top of `game.js`: `COLS`, `ROWS`, `BLOCK`, `COLORS`, `PIECES`, `LINE_SCORES`, and the initial `dropInterval`.

## Further reading

`README.md` (in Spanish) documents full gameplay mechanics, controls, and customization points in more detail.
