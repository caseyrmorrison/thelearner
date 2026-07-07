# Game Development — Simulation, the Loop, and the Engine Question

Every other program in this workspace is **event-driven**: it sleeps until a request,
keypress, or command arrives, handles it, and goes back to sleep. A game never sleeps.
It simulates a world 60+ times per second whether or not anyone touched anything, and it
must finish each frame's work in ~16.6 milliseconds, forever. That makes game
development a genuinely different discipline — soft real-time programming — and it's why
game code looks weird until the loop clicks.

This module pairs with [Rebound](../project-7-rebound-game/), a from-scratch Breakout
clone in browser canvas — no engine, no assets, no installs. The "which engine?"
question gets a real answer in §6.

---

## 1. The Game Loop — the Heartbeat

Everything is this, forever:

```
while (running) {
    input()      // sample the controls
    update(dt)   // advance the simulation by dt seconds
    render()     // draw the current state
}
```

The subtleties are all in `dt`:

- **Variable timestep** (`dt` = real time since last frame): simple, but physics becomes
  frame-rate-dependent — the ball moves differently on a 144Hz monitor than a busy
  laptop, and a lag spike teleports it through a wall.
- **Fixed timestep with an accumulator** — the professional default, and what Rebound
  implements:

```
accumulator += frameTime
while (accumulator >= FIXED_DT) {     // e.g. FIXED_DT = 1/120 s
    update(FIXED_DT)                  // simulation always advances in exact, equal steps
    accumulator -= FIXED_DT
}
render(accumulator / FIXED_DT)        // interpolate between last two states for smoothness
```

Fixed steps buy you **determinism**: the same inputs always produce the same simulation.
That's what makes physics stable, replays possible, lockstep networking conceivable —
and, closest to home, **unit tests meaningful** (Rebound's loop tests literally step
time by hand). Read Glenn Fiedler's "Fix Your Timestep" — the canonical essay; Rebound's
`loop.js` is that essay as commented code.

- **Update and render are different jobs.** Update mutates the world; render reads it
  and draws. Mixing them (moving things during draw) is the game-code equivalent of
  business logic in the route handler — same layering sin, new costume.

## 2. Game State — Machines All the Way Down

- **The game itself is a state machine**: `menu → playing ⇄ paused → game-over → menu`.
  Each state has its own update/render/input handling; transitions are explicit. Without
  this, you get the `if (!paused && !inMenu && !dying)` pyramid — the game-dev flavor of
  the nested-conditional smell.
- **Entities compose, they don't inherit.** The classic trap: `class MovingDamagingBrick
  extends DamagingBrick extends Brick...` — inheritance trees explode combinatorially.
  The industry's answer is composition: an entity is a bag of data (position, velocity,
  hitbox, hit points), and *systems* operate over whatever has the relevant data. Taken
  to its extreme this becomes **ECS (Entity-Component-System)** — data-oriented design
  that also happens to be cache-friendly (your CE background will enjoy why: structs of
  arrays, not arrays of structs). Rebound stays deliberately short of full ECS (YAGNI at
  its scale — the ADR argues it), but the composition instinct is the takeaway.

## 3. Collision Detection — Where the CS Degree Cashes In

Two phases, because n² pair-checks die fast:

- **Broad phase**: cheaply cull pairs that *can't* collide — spatial grids, quadtrees,
  sweep-and-prune. Rebound's world is small enough to skip this (measured, documented —
  the algorithms doc's "what is n?" discipline), but the backlog's particle story will
  change the math.
- **Narrow phase**: exact tests on surviving pairs. Rebound implements the two
  workhorses of 2D games — **AABB vs AABB** (axis-aligned boxes: four comparisons) and
  **circle vs AABB** (clamp the circle's center to the box = closest point; compare
  distance to radius — a genuinely elegant little algorithm). Both live in
  `engine/collision.js` with the math derived in comments.
- **Tunneling**: a fast ball can be entirely on one side of a wall this step and
  entirely past it next step — no overlap ever exists to detect. Fixes: smaller fixed
  steps (Rebound's 120Hz update is partly this), or swept/continuous collision
  (research: "swept AABB"). Knowing this failure mode exists puts you ahead of most
  hobbyists.

## 4. The Math You'll Actually Use

Less than you fear, more often than you think — all of it in `engine/vec.js`:

- **Vectors** for position/velocity/acceleration; add, scale, length, **normalize**
  (direction without magnitude).
- **Dot product** — the Swiss-army knife: projection, "is this facing that?", and
  **reflection**: a ball bouncing off a surface with normal *n* is one formula,
  `v' = v − 2(v·n)n`. Breakout is that formula plus game feel: Rebound bends the bounce
  by *where* the ball hits the paddle, because pure physics is fair but boring — a
  deliberate, documented lie for the sake of play.
- **Lerp** (linear interpolation) — smooth anything: `a + (b−a)·t`. Render
  interpolation, animations, easing all reduce to this.

## 5. Game Feel — the Unreasonable Importance of "Juice"

A technically correct game can be lifeless. "Juice" is the craft of feedback: screen
shake on impact, particles, squash-and-stretch, easing curves, sound on every event.
The same brick-break with and without juice is a different product. This is the game
version of a lesson from the practices doc: correctness is necessary, not sufficient —
and playtesting is user research (watch someone play; say nothing; take notes; despair;
improve). Rebound's Sprint 2 is essentially a juice sprint. Research: the "Juice it or
lose it" talk (Jonasson & Purho) — twenty minutes that will permanently change how you
see games.

## 6. The Engine Question — an Honest Map

"Should I learn Unity or Unreal or Godot?" is really "what am I optimizing for?"

| Path | What it is | Choose it when |
|---|---|---|
| **From scratch** (Rebound) | You write the loop, collision, rendering | Learning fundamentals. Every engine concept below will map to something you built by hand — that's *why* this workspace starts here |
| **Framework** (raylib, SDL, Pygame, LÖVE, Phaser) | Libraries: windowing, drawing, sound — loop and architecture still yours | You want to ship small 2D games while staying close to the metal; raylib+C++ would reuse your SearchLite muscles |
| **Godot** | Free, open-source engine; scene/node model; GDScript (pythonic) + C# | **The recommended first engine here.** Light (~100MB), excellent 2D, no licensing strings, thriving community. Porting Rebound to it is the Sprint 3 graduation story |
| **Unity** | The indie/mobile industry default; C# (your Stockroom experience transfers) | You want employability breadth or mobile/AR — the biggest tutorial/asset/job ecosystem; carries corporate-licensing risk (research the 2023 runtime-fee reversal — an ADR-grade cautionary tale about platform dependency) |
| **Unreal** | AAA engine; C++ plus Blueprints visual scripting | High-end 3D, or a studio career track; enormous install and learning surface — your C++ is an asset, but this is a deep second step, not a first |

Two truths to hold at once: engines are *enormously* productive (physics, rendering,
audio, editor tooling, multi-platform export — years of work, free), and engines are
opaque until you've built the primitives once. Scratch → Godot → (Unity or Unreal if a
goal demands it) is the path this curriculum bets on.

## 7. What This Module Honestly Skips

- **3D**: transforms, cameras, the GPU pipeline, shaders — a full module of its own
  someday. Research entry points: "learnopengl.com", linear algebra you already own.
- **Multiplayer**: latency hiding is genuinely hard (client prediction, rollback,
  lockstep — research "GGPO" and "1500 archers" when curious). Single-player first is
  not cowardice; it's sequencing.
- **Asset production**: art, animation, audio are separate crafts; Rebound renders
  vector shapes precisely so the code stays the subject.

## 8. Lab Track

| # | Lab | What it teaches |
|---|-----|-----------------|
| 1 | Read Rebound's `engine/` with the LEARNING_PATH — loop.js against Fiedler's essay, collision.js with pencil and paper | The loop and the math, properly |
| 2 | `node --test`, then break determinism on purpose: make `update` use real elapsed time and watch the loop tests fail; revert | Why fixed timestep is a testability decision |
| 3 | Play your own game for ten minutes with the browser DevTools performance panel open; find the frame budget | The 16.6ms discipline, measured |
| 4 | Sprint 2 stories: the juice sprint (screen shake, particles, WebAudio synth beeps), power-ups on the open feature branch | Game feel as engineering work |
| 5 | Sprint 3: port the game to **Godot** — scenes/nodes for your entities, its physics for your collision.js — and write the ADR comparing what the engine gave you vs took from you | The engine graduation, with receipts |

**Research prompts:** "Fix Your Timestep" (Fiedler); "Juice it or lose it"; ECS and
data-oriented design (Mike Acton's talk — CE-brain candy); how Godot's scene tree maps
to composition; swept AABB collision; why rollback netcode conquered fighting games.
