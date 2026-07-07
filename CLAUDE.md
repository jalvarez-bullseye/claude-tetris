# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Language

Always respond to the user in Spanish (español) in this repository, regardless of the language the user writes in. Code, identifiers, and file contents stay in English unless the user asks otherwise — this rule only governs conversational responses.

## Project overview

A classic Tetris implementation in vanilla JavaScript, HTML5 Canvas, and CSS. No dependencies, no build step, no package.json — just three files: `index.html`, `style.css`, `game.js`.

## Running the game

Open `index.html` directly in a browser, or serve it locally:

```bash
python3 -m http.server 8000
# or
npx serve .
```

There is no build, lint, or test tooling in this repo.

## Architecture

Everything lives in `game.js` (~300 lines) as top-level state and functions — no classes, no modules.

- **Board model**: `board` is a `ROWS × COLS` matrix where each cell is `0` (empty) or a piece color index (1–12).
- **Pieces**: `PIECES` defines the 7 standard tetrominoes (types 1–7) plus 5 non-standard pieces (types 8–12): `+`, `U`, and `Y` pentominoes, a `1×1` single block, and a hollow `3×3`. Rotation is done via matrix transpose+reverse in `rotateCW`, not by storing pre-rotated states; it works on any rectangular shape, not just 4×4/3×3 tetromino grids.
- **Piece selection** (`pickType`, `randomPiece`): the `+`/`U`/`Y` pentominoes and the hollow `3×3` spawn at random with low odds (`PENTOMINO_CHANCE`, `CHALLENGE_CHANCE`); the `1×1` single never spawns randomly — it's forced as the next piece via the `rewardPending` flag, which `clearLines()` sets when a Tetris (4 lines at once) is cleared.
- **Collision** (`collide`): checks board bounds and overlap with locked cells; used for movement, rotation, and ghost-piece projection.
- **Wall kicks** (`tryRotate`): after rotating, tries offsets `[0, -1, 1, -2, 2]` until a non-colliding position is found.
- **Game loop** (`loop`): driven by `requestAnimationFrame`; accumulates elapsed time in `dropAccum` and advances the piece one row once `dropInterval` is exceeded.
- **Line clearing** (`clearLines`): scans bottom-up, splices full rows out and unshifts empty rows at the top; re-checks the same row index after a splice.
- **Scoring/leveling**: `LINE_SCORES = [0, 100, 300, 500, 800]` multiplied by `level`; hard drop awards 2 pts/row, soft drop 1 pt/row. Level increases every 10 lines, which recalculates `dropInterval = max(100, 1000 - (level-1)*90)`.
- **Ghost piece**: `ghostY()` projects the current piece straight down via repeated `collide` checks; drawn with `globalAlpha = 0.2`.
- **Rendering**: `draw()` redraws the whole board canvas every frame (grid → locked blocks → ghost → current piece); `drawNext()` renders the preview piece on a separate canvas (`next-canvas`).

Control flow: `init()` resets all state → `spawn()` promotes `next` to `current` and generates a new `next` (triggers `endGame()` if the new piece immediately collides) → `requestAnimationFrame(loop)` starts the loop. Keyboard input is handled by a single `keydown` listener that dispatches on `e.code`. `togglePause()` (bound to `KeyP`) cancels/restarts the animation frame and reuses the same `overlay` element as game-over.

## Tunable constants (top of `game.js`)

`COLS`, `ROWS`, `BLOCK`, `COLORS`, `LINE_SCORES`, initial `dropInterval`, `PENTOMINO_CHANCE`, `CHALLENGE_CHANCE`. If `COLS`/`ROWS`/`BLOCK` change, update the `<canvas id="board">` `width`/`height` in `index.html` to match (`COLS × BLOCK`, `ROWS × BLOCK`).

## CI

`.github/workflows/claude.yml` and `claude-code-review.yml` invoke the `anthropics/claude-code-action`: the former responds to `@claude` mentions in issues/PR comments/reviews, the latter runs an automatic `/code-review` on every PR. Both need the `CLAUDE_CODE_OAUTH_TOKEN` secret.
