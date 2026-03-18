# Nibbles∞

A 3D reimagining of the classic Nibbles/Snake genre. Your snake moves on a flat 2D grid — but that grid is a **plane in 3D space**. Every time you eat a number, the plane rotates to a new random orientation and the camera reorients to face it head-on. Your snake's body persists in full (x, y, z) world coordinates across every previous orientation, coiling through space as you progress.

Single file. No build step. No framework. Runs in any modern browser.

---

## Play

Open `nibbles3d.html` directly in a browser. No server required.

**[Live demo →](#)** *(update with your hosted URL)*

---

## How to Play

Eat the numbered food items in sequence (1 through 9). Each number you eat grows your snake and rotates the play plane to a new orientation in 3D space. Complete a full 1→9 cycle to advance to the next level.

| Goal | Action |
|------|--------|
| Eat food | Move your head onto the numbered orb |
| Advance level | Complete the 1–9 sequence |
| Avoid | Walls · your own body · enemy residue · the enemy ball |

### Controls

**Keyboard — movement**

| Key | Direction |
|-----|-----------|
| `→` / `D` / Numpad `6` | Right |
| `←` / `A` / Numpad `4` | Left |
| `↑` / `W` / Numpad `8` | Up |
| `↓` / `S` / Numpad `2` | Down |
| `E` / Numpad `9` | Diagonal ↗ |
| `Q` / Numpad `7` | Diagonal ↖ |
| `C` / Numpad `3` | Diagonal ↘ |
| `Z` / Numpad `1` | Diagonal ↙ |

**Keyboard — game controls**

| Key | Action |
|-----|--------|
| `Space` | Pause / unpause |
| `Esc` | Quit to main menu |

**Mobile** — On-screen 8-direction D-pad (shown automatically on touch devices). Swipe gestures also work on the game canvas; diagonal swipes register when neither axis dominates.

---

## Mechanics

### The Plane System

The snake always moves on a flat N×N grid. That grid is defined in 3D space by three orthonormal vectors: `planeU` (screen-right), `planeV` (screen-up), and `planeNormal` (toward the camera). The camera always sits directly in front of the plane — `planeOrigin + planeNormal × distance` — so the view is always a clean, undistorted face-on 2D projection. Controls never need remapping: right is always right, up is always up.

When food is eaten, the plane rotates to a new random orientation centered on the snake's current head position. The camera smoothly transitions to the new face-on position. The snake's body, frozen in world space at its prior coordinates, becomes visible as a 3D trail threading through the void.

### Scoring

| Event | Points |
|-------|--------|
| Eat a `N` | `N × 10 × level` |
| Complete a level | `level × 100` bonus |

### Difficulty

Each food pickup grows the snake by `max(6, foodNum × 4)` additional segments and increases movement speed by 6ms per tick (minimum 55ms). Both effects compound continuously — by mid-game, the body fills much of the grid.

### The Enemy Ball *(Level 2+)*

Starting at level 2, a red enemy ball appears. It moves continuously in the plane's (u, v) space — always diagonal, always visible. It ricochets off the blue grid boundary and off any residue blobs it has previously deposited. Its speed is set to 88% of the snake's current movement rate, scaling up as the snake accelerates.

**Collision rules:**

| Collision | Effect |
|-----------|--------|
| Ball hits grid boundary | Ricochets · deposits residue |
| Ball hits snake body | Ricochets · deposits residue |
| Ball hits snake head | Ricochets · deposits residue · **player dies** |
| Ball hits food | Food value decrements by 1 (minimum 1) · food respawns elsewhere |
| Snake head enters residue | **Player dies** |
| Snake head enters enemy ball | **Player dies** |

Residue accumulates throughout the level and is cleared only on level completion. The ball's `(u, v)` position is preserved across plane rotations — it reappears in the same relative position on the new plane when the camera reorients.

---

## Technical Notes

**Stack:** Vanilla HTML/CSS/JS + [Three.js r128](https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js) (CDN). Everything lives in one file.

**Coordinate systems:** Game logic (movement, collision, food placement) operates in 2D grid space `(gx, gy)` — integer coordinates `[0, N-1]`. Rendering and 3D self-collision operate in world space via `THREE.Vector3`. The enemy ball operates in plane-space `(eu, ev)` — float offsets from `planeOrigin` along the plane axes.

**Performance:** Snake segments and connecting cylinders are pre-allocated in mesh pools and toggled visible/invisible each tick rather than created or destroyed. Residue blobs use a circular pool of 350 pre-allocated meshes.

**Bounce stability:** The enemy ball uses a `bounceCooldown` timer (320ms) that gates all collision responses after any bounce. This prevents re-entrance loops where a freshly-deposited residue immediately triggers another reflection on the next animation frame.

See `CLAUDE.md` for a full architecture reference intended for AI-assisted development.

---

## Project Structure

```
nibbles3d.html   — the entire game (HTML + CSS + JS, single file)
CLAUDE.md        — architecture reference for AI-assisted development sessions
README.md        — this file
```

---

## License

MIT
