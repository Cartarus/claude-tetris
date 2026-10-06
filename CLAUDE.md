# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

No build, lint, package manager, or test suite. Files are plain HTML/CSS/JS loaded directly by the browser.

- Run: open `index.html` directly, or serve the folder (`python3 -m http.server 8000`) and visit `http://localhost:8000`.
- Manual testing only. No automated tests exist, so there is no single-test command.

## Architecture

Three files, no modules, no globals beyond the script scope:

- `index.html`: DOM skeleton. Hardcodes the board canvas (`#board`, 300×600), the next-piece canvas (`#next-canvas`, 120×120), HUD spans (`#score`, `#lines`, `#level`), and the pause/game-over `#overlay`. `game.js` is loaded at the end of `<body>`.
- `style.css`: dark theme and overlay styling. Contains no game logic.
- `game.js`: all logic, in one flat script.

### `game.js` state and flow

- Config constants at the top (`COLS`, `ROWS`, `BLOCK`, `COLORS`, `PIECES`, `LINE_SCORES`). Piece shapes are square matrices whose non-zero cells hold a color index (1–7) that also indexes `COLORS`.
- Mutable state is module-level `let`s: `board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropAccum`, `dropInterval`, `animId`.
- Main loop: `loop(ts)` is driven by `requestAnimationFrame`. It accumulates `dt` into `dropAccum`; when it reaches `dropInterval` it either moves the piece down or calls `lockPiece()`. It then calls `draw()` and reschedules itself. Rendering is a full redraw of grid, locked board, ghost, and current piece on every frame.
- Locking: `lockPiece()` = `merge()` → `clearLines()` → `spawn()`. `spawn()` promotes `next` to `current`, generates a new `next`, and calls `endGame()` if the new piece collides immediately.
- Input is one `keydown` listener. `KeyP` toggles pause and is handled before the `paused || gameOver` guard. Movement and rotation are guarded by `collide()`. `tryRotate()` tries kick offsets `[0, ±1, ±2]`.
- Scoring and speed: `clearLines()` adds `LINE_SCORES[n] * level`, then recomputes `level = floor(lines/10) + 1` and `dropInterval = max(100, 1000 - (level-1)*90)`. Hard drop awards 2 per row, soft drop 1 per row.
- `init()` resets all state and is also the restart button handler. Calling it again cancels the previous `animId`.

### Invariants to keep when editing

- `index.html` canvas `width`/`height` must equal `COLS*BLOCK` × `ROWS*BLOCK`. Changing the constants in `game.js` alone breaks the layout.
- `drawNext()` assumes a 4×4 preview grid (`offX`/`offY` centering) and uses its own block size `NB = 30`.
- `drawBlock()` draws with `globalAlpha`. The ghost piece relies on alpha 0.2 and always resets alpha to 1 afterward.

### Known issue

`endGame()` is called from inside `loop()` (via `lockPiece` → `spawn`). It cancels `animId`, but `loop()` then runs `requestAnimationFrame(loop)` again. The loop keeps running after game over, so drop-timer locks continue and `clearLines()` can still change score and lines behind the overlay. Fix by returning early from `loop()` when `gameOver` is set.
