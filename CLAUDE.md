# CLAUDE.md — Nibbles ∞ (3D Edition)

## Project Overview

**Nibbles ∞** is a single-file browser game (`nibbles3d.html`) — a 3D reimagining of the classic Nibbles/Snake genre. The snake moves on a 2D grid that is itself a *plane in 3D space*. Each time the player eats a number, the plane rotates to a new random orientation in 3D and the camera reorients to face it head-on. The snake's body persists in full (x, y, z) world coordinates across all previous plane orientations. From level 2 onward, an enemy ball bounces continuously within the play plane, depositing hazard residues on impact.

There are no build tools, no bundler, no dependencies beyond Three.js (r128, loaded from CDN). The entire game — HTML, CSS, JS — lives in a single file.

---

## Architecture

### Core Invariant

> The camera **always** sits at `planeOrigin + planeNormal × computeCamDist()` looking at `planeOrigin` with `camera.up = planeV`.

This guarantees the player sees a perfect face-on 2D view at all times. `planeU` is always pixel-perfect screen-right; `planeV` is always pixel-perfect screen-up. Controls never need remapping regardless of plane orientation.

### Coordinate Systems

Three coordinate systems are active simultaneously. Understanding which system each subsystem uses is essential for any modification:

| System | Type | Range | Used by |
|--------|------|-------|---------|
| Grid space `(gx, gy)` | `int`, `int` | `[0, N-1]` | Snake movement, wall collision, food placement, food collision |
| World space | `THREE.Vector3` | unbounded | Rendering, 3D self-collision, lights, residue positions |
| Plane space `(eu, ev)` | `float`, `float` | `[−BOX, +BOX]` | Enemy ball position and velocity |

### Plane System

The active play surface is defined by three unit-length mutually orthogonal vectors:

```
planeOrigin  — 3D world anchor point (= snake head at time of last food pickup)
planeU       — screen-right axis
planeV       — screen-up axis
planeNormal  — cross(planeU, planeV), points toward camera
```

Grid cell `(gx, gy)` maps to world position via `g2w(gx, gy)`:

```js
new THREE.Vector3()
  .addScaledVector(planeU, gx - HALF)
  .addScaledVector(planeV, gy - HALF)
  .add(planeOrigin)
```

When the plane rotates, `planeOrigin` is set to the snake's current head position. The head is immediately placed at grid center `(HALF, HALF)`. The body retains its prior world-space positions and is never remapped. The enemy ball's `(eu, ev)` coordinates are also preserved unchanged — it reappears at the same relative position on the new plane.

### Camera Distance

Camera distance is computed dynamically to guarantee the full `N × N` grid is always fully visible at any screen aspect ratio:

```js
function computeCamDist() {
  const vfov = camera.fov * Math.PI / 180;
  const hfov = 2 * Math.atan(Math.tan(vfov/2) * camera.aspect);
  const minHalfAngle = Math.min(vfov, hfov) / 2;
  return ((HALF + 0.5) / Math.tan(minHalfAngle)) * 1.22; // 22% padding
}
```

The binding FOV axis (vertical on landscape, horizontal on portrait) is detected automatically. Called on resize and snapped after every camera transition.

### Game State Machine

```
'intro' → 'playing' ↔ 'transition' → 'playing'
                    ↓
                  'dead' → 'playing'  (lives > 0)
                         → 'gameover' (lives = 0)
'gameover' → 'playing'
```

- **`'intro'`** — title screen; static preview scene visible; enemy inactive
- **`'playing'`** — `setInterval(tick, speed)` running; enemy `updateEnemy(dt)` called each rAF
- **`'transition'`** — camera animating; game tick paused; input queued into `pendingDir`; enemy still moves and bounces
- **`'dead'`** — death flash animation; tick stopped; enemy paused
- **`'gameover'`** — overlay shown; everything stopped

**`paused` flag** — orthogonal to `gstate`. When `true` (only possible during `'playing'` or `'transition'`): `gameInterval` is cleared, `updateEnemy()` and `tickCamera()` return early, and movement key events are dropped. Toggled by `togglePause()` (Space); reset to `false` by `initGame()` and `quitToIntro()`.

---

## Key Constants

| Name | Value | Purpose |
|------|-------|---------|
| `N` | `19` | Grid cells per axis (must be odd — center = `HALF, HALF`) |
| `HALF` | `9` | Grid center index; also enemy spawn reference |
| `BOX` | `9.5` | Enemy bounding half-extent in plane-space (`HALF + 0.5`) |
| `CAM_DUR` | `0.38s` | Camera transition duration |
| `POOL` | `400` | Pre-allocated snake segment + cylinder meshes |
| `RESIDUE_POOL` | `350` | Pre-allocated residue sphere meshes (circular buffer) |
| `MARGIN` | `2` | Food placement inner border (cells from edge) |
| `ENEMY_SPD_FRAC` | `0.88` | Enemy speed as fraction of snake's current step rate |
| `R_HIT` | `0.52` | Enemy → residue collision radius (world units) |
| `S_HIT` | `0.50` | Enemy → snake segment collision radius (world units) |
| `HEAD_R_HIT` | `0.46` | Snake head → residue death radius (world units) |

---

## Mechanics

### Movement

8 directions: 4 cardinal + 4 diagonal. `dir` and `nextDir` are `{gx, gy}` objects where each component is `−1`, `0`, or `1`. U-turn is blocked: `gx === −dir.gx && gy === −dir.gy`. During `'transition'`, input is buffered into `pendingDir` and applied in `onTransitionComplete()`.

On food pickup, the current movement direction is projected onto the new plane to pick the closest-matching axis and preserve momentum:

```js
const movWorld = planeU.clone().multiplyScalar(dir.gx).addScaledVector(planeV, dir.gy);
const proj = movWorld.clone().addScaledVector(newN, -movWorld.dot(newN));
// pick axis in {±newU, ±newV} with highest dot against proj
```

### Collision Priority in `tick()`

Per-tick collision checks run in this order before the move is committed:

1. **Wall** — `nGX < 0 || nGX >= N || nGY < 0 || nGY >= N` → `die()`
2. **Residue** — `nWorld.distanceTo(r.pos) < HEAD_R_HIT` for all residues → `die()`
3. **Enemy ball** — `nWorld.distanceTo(enemyWorld()) < S_HIT + 0.1` → `die()`
4. **Self** — `snake[i].distanceTo(nWorld) < 0.1` for `i < snake.length − 1` → `die()`
5. **Food** — `nGX === foodGX && nGY === foodGY` → eat

### Growth & Speed

Each food pickup grows the snake by `Math.max(6, foodNum × 4)` segments (eating a `4` adds 16 segments). Speed increases by 6ms per pickup, floored at 55ms. Both ramp continuously — no reset between plane rotations.

### Plane Rotation

`randomPlane()` generates a new orthonormal basis `(n, u, v)` differing from the current normal by at least 25° (`|dot| < 0.75`). `v` is biased upward (`v.y >= 0`) to prevent disorienting upside-down flips.

### Food Placement

Food is restricted to the inner margin `[MARGIN, N−1−MARGIN]` on both axes (currently `[2, 16]`), keeping it always fully visible after any plane rotation. The blocked set includes all snake world-positions projected onto the new plane grid.

---

## Enemy Ball

### Design

The enemy ball exists entirely in plane-space `(eu, ev)` — float offsets from `planeOrigin` along `planeU`/`planeV`. Its world position is derived on demand:

```js
function enemyWorld() {
  return new THREE.Vector3()
    .copy(planeOrigin)
    .addScaledVector(planeU, enemy.eu)
    .addScaledVector(planeV, enemy.ev);
}
```

Because the enemy stores `(eu, ev)` rather than a world position, plane rotations are free — the ball automatically re-projects onto the new plane without any explicit remapping. It is structurally impossible for it to appear outside the grid after a rotation.

### State

```js
enemy = {
  eu, ev,          // plane-space position
  vu, vv,          // plane-space velocity (units/sec)
  active,
  hitCooldown,     // blocks death re-triggering after a head strike (0.45s)
  bounceCooldown,  // blocks ALL collision responses after any bounce (0.32s)
}
```

### Speed

Speed is set dynamically to `(1000 / speed) * ENEMY_SPD_FRAC` — 88% of the snake's current step rate in world-units/second. Recalculated each frame so the enemy always accelerates in lockstep when the snake speeds up.

### Bounce Stability — `bounceCooldown`

This is the most important implementation detail of the enemy system. Without it, the ball freezes.

**The freeze mechanism:** Any bounce deposits a residue near the ball's position. On the next animation frame (~16ms), the ball has moved only `vu × 0.016 ≈ 0.07` units — still inside `R_HIT = 0.52` of the fresh residue. The residue-bounce fires again, reflecting velocity back. The ball oscillates between two adjacent residues indefinitely.

**The fix:** `bounceCooldown = 0.32` is set after *any* collision response (wall, residue, snake, food). All collision detection is gated behind `bounceCooldown <= 0`. During those 320ms the ball travels ~2.2 units and exits the collision zone cleanly before detection resumes.

Do not replace `bounceCooldown` with position-based nudges — they are insufficient at high speeds and re-introduce the freeze at frame-rate boundaries.

### Collision Rules

| Trigger | Effect |
|---------|--------|
| Ball reaches `±BOX` boundary | Reflect `vu`/`vv` on that axis · deposit residue · `bounceCooldown = 0.32` |
| Ball within `R_HIT` of any residue | 2D reflect in plane-space · nudge clear · deposit residue · `bounceCooldown = 0.32` |
| Ball within `S_HIT` of snake body (`i > 0`) | 2D reflect · nudge clear · deposit residue · `bounceCooldown = 0.32` |
| Ball within `S_HIT` of snake head (`i = 0`) | Same + `die()` if `hitCooldown <= 0` |
| Ball within `0.70` of food | `foodNum = max(1, foodNum − 1)` · rebuild sprite · `placeFood()` · `bounceCooldown = 0.32` |
| Snake head moves into residue | `die()` (checked in `tick()`, not `updateEnemy()`) |
| Snake head moves into enemy ball | `die()` (checked in `tick()`, not `updateEnemy()`) |

### Residue Pool

Residues are stored as `THREE.Vector3` world positions in the `residues` array. Their meshes come from `residuePool[]` — a circular buffer of 350 pre-allocated spheres. When the buffer wraps, the oldest residue's mesh is reassigned; the old `residues` entry is updated in place if its `idx` is recycled.

Residues persist through plane rotations (world-space positions are unchanged). They are cleared only on `levelUp()` or full game reset.

---

## Rendering

### Mesh Pools

Snake segments and cylinders are pre-allocated (`segPool`, `cylPool`, size `POOL = 400`). `refreshSnake(hide)` iterates the full pool each call, setting `visible` and mutating material properties in place. No mesh creation or disposal occurs at runtime.

### Connecting Cylinders

```js
_qt.setFromUnitVectors(new THREE.Vector3(0,1,0), diff.normalize());
cyl.quaternion.copy(_qt);
cyl.scale.y = dist * 0.85;
cyl.position.copy(a).add(b).multiplyScalar(0.5);
```

### Food Sprite

The food number is rendered to a `128×128` `<canvas>` and uploaded as a `THREE.CanvasTexture` on a `THREE.Sprite`. Recreated via `makeFoodSprite(num)` whenever `foodNum` changes (on eat or enemy hit). Sprite quaternion synced to `camera.quaternion` each rAF so it always faces the camera.

### Grid

Rebuilt via `rebuildGrid()` on every plane rotation. Three `LineSegments` objects: faint inner cell grid, bright outer border (`0x0055bb`), and a faint center crosshair. All attached to `gridGroup`, which is fully cleared (with geometry disposal) before each rebuild.

### VFX Ring Bursts

`spawnRingBurst(pos, normal, color, count, dur, endScale)` emits `count` expanding `RingGeometry` meshes from a pool of 30 pre-allocated instances (`_vfxPool`). Each ring is oriented to face `normal` via `Quaternion.setFromUnitVectors`, staggered 60ms apart, and driven by `_vfxActive` tweens updated in `updateVFX(now)` each rAF. Opacity follows `0.85 × (1 − t²)`; scale lerps from `startScale` to `endScale`. Active meshes are tracked in `_vfxBusy` (a `Set`) to prevent double-allocation.

Triggered at:
- **Snake head death** (`die()`): white (`0xffffff`), 3 rings, 450ms, maxScale 4.5 — fires before the flash animation
- **Enemy captures food** (`updateEnemy()`): orange-red (`0xff4400`), 3 rings, 500ms, maxScale 4.0 — fires at the food's world position

### Enemy Visuals

Three objects track `enemyWorld()` each frame: `enemyMesh` (solid red sphere), `enemyGlowMesh` (additive backface sphere), `enemyLight` (point light). Six trail spheres (`trailMeshes`) sample `trailHistory` at intervals of 3 frames, fading from `0.35` to near-zero opacity.

---

## Controls

### Movement

Keyboard uses two maps checked in order per `keydown`. Arrow/WASD/Q/E/Z/C are looked up by `e.key` in `KEY_MAP`. Numpad keys are looked up by `e.code` in `NUMPAD_MAP` — `e.code` is used so they work regardless of NumLock state.

| Input | Direction |
|-------|-----------|
| `→` / `D` / Numpad `6` | `+gx` |
| `←` / `A` / Numpad `4` | `−gx` |
| `↑` / `W` / Numpad `8` | `+gy` |
| `↓` / `S` / Numpad `2` | `−gy` |
| `E` / Numpad `9` / d-pad ↗ | `+gx, +gy` |
| `Q` / Numpad `7` / d-pad ↖ | `−gx, +gy` |
| `C` / Numpad `3` / d-pad ↘ | `+gx, −gy` |
| `Z` / Numpad `1` / d-pad ↙ | `−gx, −gy` |
| Swipe | Cardinal or diagonal (diagonal when `0.4 < |dy/dx| < 2.5`) |

### Game Controls

| Key | Action |
|-----|--------|
| `Space` | Pause / unpause (`togglePause()`). Clears `gameInterval` on pause; restarts it on unpause if `gstate === 'playing'`. Also starts/retries from overlay screens. |
| `Esc` | Quit to intro (`quitToIntro()`). Stops the interval, hides all overlays, shows `ov-intro`, sets `gstate = 'intro'`. No-op while already on intro. |

Movement keys are ignored while `paused === true`. `paused` is reset to `false` on every `initGame()` call.

D-pad shown via `(hover: none) and (pointer: coarse)` and `max-width: 600px`.

---

## Extension Notes

**Adding a second enemy:** `enemy` is a plain object. Promote it to an array `enemies[]` and run `updateEnemy(dt, e)` for each. `bounceCooldown` must be per-instance.

**Adding multiplayer:** Plane state is fully described by `(planeOrigin, planeU, planeV, planeNormal)` plus the snake array — all serializable to JSON. Enemy `(eu, ev, vu, vv)` is also trivially serializable.

**Saving replays:** `snake` is a `THREE.Vector3[]` appended each tick. It fully describes the 3D path and can be replayed by re-running `g2w()` calls against recorded plane states.

**Adding obstacles:** Extend the `blocked` array in `placeFood()` to mark obstacle cells. Any world-space object on the plane can be added by projecting through `g2w()`.

**Changing grid size:** `N` must stay odd. Increasing it requires adjusting `MARGIN`. `BOX = HALF + 0.5` updates automatically. Recheck the `computeCamDist()` padding multiplier if content clips.

**Performance:** The main cost is `refreshSnake()` — O(POOL) material mutations per tick. For very long snakes (>200 segments), switch to `InstancedMesh` with per-instance color via `setColorAt()`. The residue pool is already circular and bounded.