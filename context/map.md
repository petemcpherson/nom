# NOM — Architecture Map

Quick orientation for AI agents. Broad overview only; follow the line pointers into `index.html` for details.
Line numbers reflect `index.html` at time of writing (~885 lines) — they drift as code changes, so search by function name if they don't match.

## What it is

NOM is a single-file browser arcade game (agar.io-like). You steer a yellow blob with the mouse, eat smaller blobs, flee bigger ones, dodge spikes, and grow through "stages" (Petri Dish → Puddle → … → The Void). The end of each new stage is guarded by a boss; beating it advances you to the next stage and grants a roguelite upgrade pick.

## Tech stack

- Plain HTML + vanilla JavaScript + Canvas 2D. No framework, no build step, no dependencies, no package.json.
- Audio: Web Audio API oscillators (procedural blips, no audio files).
- Persistence: `localStorage` key `nom-best2` (best score only).
- Run: open `index.html` in a browser (or serve the folder with any static server).
- No tests, no linting config.

## Repo layout

```
/index.html        Entire game: <style>, <canvas id="c">, one inline <script>
/context/map.md    This file
```

## Code layout inside `index.html`

The script is organized by `// ---------- section ----------` comment headers, in this order:

| Section | Lines (approx) | Contents |
|---|---|---|
| setup | 15–26 | canvas `c`, context `g`, `W`/`H`/`DPR`, `resize()` (DPR capped at 2) |
| input | 28–36 | `mouse` {x,y} in screen coords; pointer + keyboard → `onTap()` / `choose()` |
| audio | 38–54 | `blip()`, `arp()` (note sequence), `boom()` (death sound) |
| stages | 56–73 | `STAGES` array, `stageOf`, `stageName`, `TIER_GROW`, `tierR`, `targetScale` |
| state | 75–78 | `rand()`, `best` loaded from localStorage |
| upgrades & bosses (data) | 80–100 | `UPGRADES` and `BOSSES` definition tables |
| globals + `reset()` | 102–123 | all mutable game state; `reset()` initializes a run |
| spawning / input actions | 125–190 | `viewR`, `spawn`, `onTap` (start/dash/pick), `startPick`, `choose`, `cardRects`, `inRect` |
| bosses (logic) | 192–324 | `spawnBoss`, `updateBoss`, `hitBoss`, `chunk`, `killBoss`, `updateShots` |
| update | 326–503 | `update(dt)` main sim, `eat`, `hurt`, `die`, `burst` |
| draw | 505–870 | `draw()` plus helpers: `blobPath`, `spikePath`, `drawEyes`, `fillBg`, `blobColor`, `drawPick`, `wrap`, `edgeArrow`, `drawBoss`, `big`, `small` |
| loop | 872–882 | `frame()` via `requestAnimationFrame`; `dt` clamped to 0.05s |

## Core concepts

### Game modes (state machine)

`mode` is a global string: `'title' | 'play' | 'pick' | 'dead'`.

- `title` → tap → `reset()` + `play`
- `play` → growing past the end of the newest stage → player is held at the threshold and `spawnBoss()` runs
- `killBoss()` → sets `cleared`, and after `clearT` (1.8s) bumps `player.r` over the threshold → tier up → `startPick()` → `pick`
- `pick` → `choose(i)` → `play`
- `play` → `die()` → `dead` (unless `up.lives > 0`, which consumes a spare life instead)
- `dead` → tap after 0.8s → `reset()` + `play`

While in `pick`, `update()` only advances `pickTimer` (the world is frozen). Picks are ignored for the first 0.5s to prevent accidental clicks.

Pause is a separate `paused` flag, not a mode. P or Escape toggles it during `play`. Any tap resumes without dashing. While `paused` is set, `update()` returns at once and `draw()` shows a PAUSED overlay. `reset()` clears the flag.

### World & camera

- World coordinates are unbounded and centered on the player. The player is always drawn at screen center.
- `camS` is the world→screen scale. It eases toward `targetScale()`, which depends on the current `tier`, so the camera zooms out in discrete steps as you change stage.
- `toS(wx, wy)` (local inside `draw`) converts world coordinates to screen coordinates.
- `viewR()` is the world-space radius of the visible area. It drives spawn distances, AI aggro ranges, and despawn (`> 2.2 * viewR`).
- Speeds are authored in screen pixels and divided by `camS` (`/ s`), so motion feels the same at every zoom level.

### Tiers / stages

- `tier` is the current stage index; `maxTier` is the highest tier reached this run.
- `tierR(t) = 10 * 2.5^t` is the baseline radius for tier `t`.
- Tier up happens when `player.r > tierR(tier) * TIER_GROW`. Tier down happens when `player.r < tierR(tier) * 0.7`.
- At the newest tier (`tier === maxTier`), tier up is gated: `player.r` is clamped to the threshold and a boss spawns. Only after `killBoss()` (`cleared`, then `clearT` runs out) does the tier advance.
- Only a *new* `maxTier` gives a banner, points, and an upgrade pick. Re-entering an old tier after shrinking has no gate.
- Past the last `STAGES` entry, `stageName` appends a number ("The Void 2", …).
- The background crossfades from `bgFrom` to `tier` using `bgT`.

### Player growth / metabolism

- Eating combines area: `r = sqrt(r² + b.r² * up.growth)`.
- Constant shrink: `r *= 1 - up.shrink * dt`. Dying with `r < 5` gives the message "shrank to nothing".
- `hurt()` shrinks the player by `up.hurtLoss`, gives 1.2s of `invuln`, resets combo, and applies knockback.

### Dash

- Tap/click/Space dashes toward the mouse (`onTap`). This costs one of `dashCharges`, up to a max of `up.charges`.
- `dashT > 0` means the player is dashing. `dashCd` is the recharge timer. A ring around the player shows recharge progress.
- Dashing into a boss (`dashT > -0.05`) damages it. Plain contact with a boss hurts the player.

### Combo & score

- Each bite increments `combo` and refreshes `comboTimer = up.comboTime`. Points per bite = `b.r * combo`.
- Other score sources: tier-up `500*tier`, boss hit `100*b.lvl`, boss kill `2000*b.lvl`.

### Powers (from gold blobs)

`power = { kind: 'FRENZY' | 'MAGNET', t, dur }`.
- FRENZY: every non-spike blob is edible.
- MAGNET: edible blobs in range get pulled toward the player.
Duration is `up.powerDur`.

## Data structures

```js
player = { x, y, r, vx, vy, wob }

blob = {                      // entries in `blobs`
  x, y, r, vx, vy,
  type: 'normal' | 'gold' | 'spike' | 'dasher',
  ph,                         // wobble phase (animation)
  eyes: bool,
  st: 'idle' | 'charge' | 'dash',   // dasher AI state
  t,                          // AI timer
  edible,                     // recomputed each frame in update()
}

boss = {                      // single global, or null
  ...BOSSES[i],               // kind, name, hue, desc
  lvl,                        // tier + 1: the stage this boss unlocks; drives hp, speed, score
  x, y, r, vx, vy, hp, maxHp,
  st, t, stun, hitCd, flash, ph, shootT,
}

shot     = { x, y, vx, vy, r, life }          // `shots`, boss projectiles
particle = { x, y, vx, vy, r, hue, life }     // `parts`
text     = { x, y, t, life, big? }            // `texts`, floating text in world space
banner   = { text, sub, t }                   // screen-centered announcement
power    = { kind, t, dur }

up = { ...BASE_UP }           // upgrade-modified stats, reset each run
  // dashCd, charges, chargeBits, shrink, growth, hurtLoss, gold,
  // powerDur, magnet, comboTime, bossDmg, speed, lives, lifeBits
taken = { [upgradeId]: count }               // pick counts; shown as Lv on cards and the build line
choices = [UPGRADE, ...]                     // up to 3 cards while in `pick`
```

## Content tables (edit these to add content)

- **`STAGES`** (~line 57): `{ name, a, b }`, where `a`/`b` are the inner/outer radial-gradient colors.
- **`UPGRADES`** (~line 81): `{ id, name, desc, apply(up), now(up) }`. Upgrades are deliberately tiny (3–8% per pick) and uncapped, so a run's build stacks up incrementally. `now(up)` renders the running total on the card. Discrete stats use fractional progress: `chargeBits` (every 3rd Dash Cell adds a charge) and `lifeBits` (every 4th Nine Lives shard adds a life). To add an upgrade, add a base stat to `BASE_UP`, then read it where it applies.
- **`BOSSES`** (~line 95): `{ kind, name, hue, desc }`. The boss guarding the end of tier `t` is `BOSSES[t % 4]`. Per-kind behavior is in `updateBoss` (`splitter`, `spitter`, `vortex`, `charger`) and `drawBoss` (vortex rings). `drawBoss` draws every boss with an aura, rotating spike ring, horns, slit eyes and a toothed grin that gapes during a charge. Splitter also gets extra chunks in `hitBoss`.
  - Boss HP = `round(4 + lvl*1.5)`. Boss radius = `4 * player.r` at spawn. From `lvl` 5 on, every boss shoots.

## Entity behavior summary (`update`, ~lines 381–432)

- **Spawn mix** (`spawn`): gold uses `up.gold` chance. Spikes appear from tier ≥ 1, dashers from tier ≥ 2. Otherwise a normal blob that is mostly smaller than the player. Blob size = `player.r * k`.
- Population is kept at 45 blobs (32 during a boss fight). Blobs beyond `2.2*viewR` despawn.
- **Edible** (smaller than `0.98*player.r`, or FRENZY): flee slowly, or get pulled in by magnet.
- **Bigger normal**: hunts the player. **Dasher**: idle → charge (shakes) → dash at 780 px/s.
- **Spike**: slow drift. Contact hurts the player (does not kill).
- **Gold**: flees fast. Eating it grants a random power and +3 combo.
- Contact with a bigger blob kills the player ("got NOMMED" / "got rammed").

## Rendering notes (`draw`, ~line 571)

Draw order: screen shake translate → bg gradients → parallax stars → magnet rings → blobs → boss → shots → particles → player (dash ring, squish, eyes, mouth) → off-screen threat arrows (`edgeArrow`) → floating texts → white flash → HUD (score, stage, progress bar, combo, power bar, boss HP) → banner → mode overlays (title / dead / `drawPick`).

- Color is HSL hue per blob from `blobColor()`: teal = edible, red = danger, orange = dasher, purple = spike, gold = gold.
- `cardRects()` lays out the upgrade cards. Wide screens (>720px) get a horizontal row. Narrow screens get a vertical stack. The same rects are used for both drawing and hit-testing.

## Common change recipes

- **Tune difficulty**: `spawn()` probabilities; `huntPull`/`huntMax` in `update`; boss `hp`/speed in `spawnBoss`/`updateBoss`; base `up` values in `reset()`.
- **New blob type**: branch in `spawn`, AI branch in the `update` blob loop, collision branch, and colour in `blobColor`.
- **New boss**: entry in `BOSSES`, plus a `kind` branch in `updateBoss` (and optionally `drawBoss`).
- **New power**: set `power.kind` in `eat()`, read it in `update`/`draw`, and add a HUD label in `draw`.
- **Reset best score**: change the `nom-best2` key, or clear localStorage.
