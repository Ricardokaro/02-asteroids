# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A clone of the classic arcade game Asteroids, built with plain HTML5 Canvas and vanilla JavaScript (ES6+). No frameworks, no bundler, no package manager, no dependencies. The entire game logic lives in a single file, `game.js`, loaded directly by `index.html`.

## Running the game

Open `index.html` directly in a browser, or serve it locally:

```bash
npx serve .
```

There is no build step, no test suite, and no linter configured — changes to `game.js` or `index.html` take effect on page reload.

## Architecture

Everything is in `game.js`, organized top-to-bottom as:

- **Input** — a global `keys` map (currently-held keys) and `justPressed` map (single-frame edge detection via the `pressed(code)` helper), populated by `keydown`/`keyup` listeners.
- **Utils** — `wrap` (toroidal position wrapping), `dist`, `rand`, `randInt`.
- **Entity classes** — `Bullet`, `Asteroid`, `Ship`, `Particle`. Each follows the same shape: constructor sets initial position/velocity, `update(dt)` advances physics and sets `this.dead` when expired, `draw()` renders via the shared `ctx`. There is no base class — the contract is duck-typed.
- **Game state** — module-level `let` variables (`ship`, `bullets`, `asteroids`, `particles`, `score`, `lives`, `level`, `state`, `deadTimer`) rather than a state object or class. `state` is one of `'playing' | 'dead' | 'gameover'`.
- **`update(dt)`** — the core game loop step: branches on `state` first (gameover/dead have their own short-circuited update logic), otherwise updates all entities, then does collision detection (bullet↔asteroid, ship↔asteroid) using simple circle-distance checks against each entity's `radius`, and advances the level when `asteroids.length === 0`.
- **`draw()`** — clears the canvas and renders particles → asteroids → bullets → ship → HUD, in that order (back to front).
- **Main loop** (`loop(ts)`) — a `requestAnimationFrame` loop computing `dt` in seconds, clamped to 0.05s max to avoid physics blowups after tab-switch/lag, then calling `update(dt)` then `draw()`.

### Key gameplay constants worth knowing before tuning behavior

- World is toroidal: everything wraps at `W`/`H` (800×600) via `wrap()`.
- Asteroids have 3 sizes (`size` 3→1, large→small) with per-size arrays for radius (`RADII`), speed (`SPEEDS`), and score value (`POINTS`); splitting an asteroid (`Asteroid.split()`) spawns two of `size - 1` at the same position, size 1 splits into nothing.
- Ship movement is thrust + drag based (`THRUST`, `DRAG`, `ROT` constants in `Ship.update`), not directly velocity-set.
- Ship has temporary invincibility on spawn/respawn (`invincible` timer), visualized as blinking in `draw()`.
- Bullets have a `ttl` (time-to-live) rather than being cleaned up by leaving the screen (they wrap instead).

### Adding a new entity type

Follow the existing class pattern: constructor for initial state, `update(dt)` mutating position/state and setting `this.dead = true` when it should be removed, `draw()` using `ctx` directly. Wire it into the module-level state arrays, into `update()`'s per-frame loop and filtering (`array = array.filter(x => !x.dead)`), and into `draw()`'s render order.
