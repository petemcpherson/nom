# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

NOM is a single-file browser arcade game (agar.io-like roguelite): steer a blob with the mouse, eat smaller blobs, grow through stages, pick an upgrade and fight a boss at each new stage.

**Read `context/map.md` first.** It is the detailed architecture map (state machine, camera/tier math, data structures, content tables, change recipes). Keep it in sync when you change structure; its line numbers drift, so search by function name.

## Running

- No build, no dependencies, no package.json, no tests, no linter.
- Run: open `index.html` in a browser, or serve the folder with any static server (e.g. `python3 -m http.server`).
- Verify changes by playing in the browser; check the console for errors.

## Architecture essentials

- Everything lives in `index.html`: `<style>`, `<canvas id="c">`, and one inline `<script>` of vanilla JS + Canvas 2D. Audio is procedural Web Audio (no asset files).
- The script is split by `// ---------- section ----------` headers: input, audio, stages, state, upgrades & bosses (data), bosses (logic), update, draw, loop. Add code to the matching section.
- All game state is module-level globals, initialized in `reset()`. `mode` (`'title' | 'play' | 'pick' | 'dead'`) drives the state machine; in `pick` the world is frozen.
- World is unbounded and player-centered. `camS` is the world→screen scale and eases toward `targetScale()` per tier. Speeds are authored in screen pixels and divided by `camS`, so motion feels the same at every zoom. `viewR()` drives spawn, aggro, and despawn distances.
- Tier radius is `tierR(t) = 10 * 2.5^t`; only a new `maxTier` triggers banner, upgrade pick, then boss.
- Run-scoped upgrade stats live in the `up` object; upgrades mutate it via `UPGRADES[i].apply(up)`. New upgrades usually need a new `up` field in `reset()` plus a read site.
- Content is data-driven: `STAGES`, `UPGRADES`, `BOSSES` tables. Boss behavior branches on `kind` in `updateBoss` / `drawBoss`.
- `cardRects()` is shared by upgrade-card drawing and hit-testing — change both behaviors through it.
- Best score persists in `localStorage` key `nom-best2`.
