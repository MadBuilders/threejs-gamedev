# Physics

## Objective
Integrate physics into Three.js games pragmatically, avoiding letting the physics engine swallow the architecture or overcomplicate gameplay.

## Before adding an engine

Three.js **does not have collision detection**. It is a rendering library. What it offers are geometric helpers (`Raycaster`, `Box3`, `Sphere`, `Plane`, `Frustum`) with their `intersect*` methods, which are math utilities, not a collision system.

That means the first decision is not "which physics engine do I use", but **"do I need an engine?"**. In most casual/lightweight 3D games, the answer is no.

Strategy ladder, from lowest to highest cost:

1. **Point raycasting**. `THREE.Raycaster` against a mesh or list of meshes. Ideal for picking (click), line-of-sight, terrain height under the player, weapon hit tests. Cheap if the targets are few or the mesh is accelerated with BVH.
2. **Manual bounding volumes**. Your own list of colliders (`{ center, radius }` for cylinders/spheres in XZ, `Box3` for AABB). Hand-written tests per frame (`dx*dx + dz*dz < r*r`, or `Box3.intersectsBox`). Enough for 80% of casual games: walking games, top-down, lightweight shooters, simple multiplayer. No WASM, no step, no broadphase — just math.
3. **BVH over static mesh**. With [`three-mesh-bvh`](https://github.com/gkjohnson/three-mesh-bvh) when you really need queries against complex world geometry (raycasts against walls in a carved level, shapecasts for precise movement). Sub-ms for large geometries.
4. **Real physics engine** (Rapier, Cannon-es, Ammo). When there are **forces, persistent contacts, automatic resolution between bodies**, or constraints. Then it is worth paying the cost.

Choose the lowest rung that solves the problem. Move up only when the previous rung falls short, not out of inertia.

## The collider is never the visual asset

Hard rule: the GLB mesh (the pretty model with thousands of tris, normals, PBR, skinning) is **not** the collider. Confusing them is a classic and expensive anti-pattern.

Reasons:
- Collision against arbitrary geometry is extremely expensive compared to a capsule or AABB.
- The mesh can contain holes, weird normals, silent `doubleSided: true` (very common in AI-generated GLBs), degenerate triangles. All of that produces collision bugs that are hard to debug.
- An artist or external tool can re-export the GLB with more detail and suddenly your collision breaks or becomes expensive without warning.
- Runtime needs a **stable and predictable** representation: a radius, a capsule, a box. The GLB does not provide that by itself.

Correct pipeline:
- The **visual asset** lives in the GLB.
- The **logical collider** is declared by you (trunk radius for trees, AABB for houses, capsule for characters). It can be a hardcoded number per variant, a value calculated once from the mesh bounding box on load, or a collider authored in the level editor.
- Both share transform (position/rotation/scale), but they are different representations.

This applies to all four rungs in the previous section. It is true whether you do collision by hand or add Rapier: **never** pass the GLB trimesh as the default collider.

## Main rule
Use physics only where it adds real value.

Not every object needs full simulation. Often this is enough:
- simple collisions
- triggers
- kinematic movement
- targeted constraints

## Healthy separation
Three.js renders. The physics engine simulates.

Do not mix those responsibilities.

Think in three layers:
1. **visual representation**
2. **physical representation**
3. **synchronization between the two**

## Initial recommendation
Have a physics layer or bridge that:
- creates bodies and colliders
- advances the simulation
- synchronizes visual transforms
- exposes useful gameplay events or queries

## When to use full physics
It is usually worth it for:
- objects that truly fall, bounce, or push
- systemic interaction between several bodies
- vehicles or mechanisms if the game depends on them
- gameplay based on balance, forces, or emergent collisions

## When it is NOT needed
It is often not needed for:
- simple triggers
- pickups
- static obstacles
- doors or platforms with scripted movement
- basic proximity detection

## When switching to a physics engine is worth it

Operational question for deciding whether to add Rapier (or equivalent) to a project that currently solves collision by hand: **does the mechanic need something I would not write in 50 lines?**

Concrete checklist. If any of these appear, Rapier stops being overhead and starts being savings:

- **Vehicles with wheels and suspension**. A raycast vehicle controller is very hard to clone well by hand; Rapier includes it.
- **Body stacking** (boxes, barrels, collapsing towers). Resolving multiple contacts by hand is a bottomless pit.
- **Real slopes with continuous contact** where the player must stick to the ground without clipping through triangles or hopping on mesh edges.
- **Physical projectiles with bounce, friction, and emergent effects** (not straight bullets — those are raycast).
- **Joints and constraints** (hinged doors, chains, suspension bridges, ragdolls).
- **Emergent systemic mechanics** where the design *depends* on bodies interacting with each other without scripting (physics puzzles, dominos, pushing objects into zones).

Opposite signal (probably not needed):
- Characters that only need to avoid passing through obstacles and collide with each other → manual cylinders/capsules in XZ.
- Pickups, triggers, zones → AABB against AABB, no engine.
- Flat level or smooth terrain without sharp physical contact → `getGroundHeight(x, z)` and stay kinematic.
- Shots or line-of-sight → `Raycaster`, not bodies.

Corollary: **Rapier when the interaction calls for it**. Do not add it "just in case" or "because every serious 3D game has physics". Each rung of the strategy ladder has its niche.

## Key principle
Do not use a physics engine to solve a problem you already understand better with game logic.

Realistic physics does not always produce the best game.

## Body types and practical use

### Static
For floors, walls, fixed world.

### Dynamic
For objects that should react to forces and collisions.

### Kinematic
For characters, platforms, or elements governed by custom logic that still interact with the physical world.

## Colliders
Prefer simple colliders whenever possible:
- boxes
- spheres
- capsules
- approximate planes

Do not use complex mesh as the default collider unless there is a real need.

## Player controller
The main character usually needs special care.

Recommendations:
- do not fully depend on raw physics for player control
- separate movement intent from physical response
- define ground, jump, slope, and contact clearly
- avoid making the character feel "gelatinous" in pursuit of realism

Strong pattern that performs very well in real examples:
- kinematic player
- simple collider, usually capsule
- world queried through octree, queries, or static colliders
- gravity, grounded state, and jump resolved by the controller

That usually gives a much more controllable result than dropping in a humanoid rigid body and praying.

## Timing and update
Physics needs a clear rhythm.

Useful rules:
- use a controlled simulation step
- do not let variable delta break stability
- synchronize visuals after the physics step
- clearly document update order

## Performance
Physics also consumes a lot.

Watch:
- number of active bodies
- number of complex colliders
- query frequency
- continuous collision cost
- sleeping objects that might not update

### Concrete Rapier cost

When Rapier is added, the real cost is distributed like this (order of magnitude to keep in mind):

- **Bundle**. `@dimforge/rapier3d-compat` weighs ~1–1.5 MB. The `compat` version inlines WASM in JS to avoid pain with Vite/ESM. This is noticeable on mobile boot.
- **Async init**. `await RAPIER.init()` before creating the world. Hundreds of ms on low-end mobile. Not per-frame, but it delays "first playable". Start it in parallel with asset loading when possible.
- **`world.step()` per frame**. With dozens of kinematic bodies and primitive colliders, sub-ms on desktop. Scales with:
    - active **dynamic** bodies (kinematics and statics are almost free)
    - contact pairs (the broadphase filters the rest — total colliders matter less than how many are near each other)
    - collider type: cuboid/ball/capsule are cheap; **large trimesh and heightfield** colliders are heavy in queries and memory
- **Rapier ↔ Three.js synchronization**. Every frame, read `body.translation()` / `body.rotation()` and copy them into the `Object3D`. Done badly (new objects per frame, `.clone()` in the hot path), this causes visible GC. Use module-level scratch vectors/quaternions.
- **Fixed timestep is mandatory**. Rapier expects fixed `dt` (typically 1/60). Passing the variable `rAF` `dt` directly causes instability *and* more substeps = more cost. Correct pattern: accumulator with fixed step, render interpolation separately.
- **Queries (raycast, shapecast)**. Cheap per call, but spamming them every frame for every NPC adds up. Share results and cache where it makes sense (ground check may come from the body's own step instead of an extra raycast).

Mental numbers (decent desktop, web singleplayer game):

- 50–200 kinematics + a handful of dynamics + primitive colliders → **<1 ms/frame** for `world.step()`. Invisible.
- 500+ interacting dynamics → **2–5 ms**. Noticeable, still acceptable.
- Whole-level trimesh + per-NPC raycasts without culling → **10–20 ms spikes** and stutter. That is where it really hurts.

Practical conclusion: Rapier used well is very fast. "Rapier is slow" almost always means "Rapier is badly integrated" — see specific anti-patterns below.

## Gameplay and physics
Physics should serve the design, not dominate it.

Useful questions:
- does this improve game feel?
- is it more fun, or just more realistic?
- can I get a better result with simpler logic?

## Anti-patterns
- adding physics to everything out of inertia
- using complex colliders for everything
- coupling gameplay directly to physics-engine callbacks
- not distinguishing visual body from physical body
- trusting realistic physics to fix bad control design

## Rapier-specific anti-patterns

Most cases of "Rapier is slow" or "Rapier has weird problems" in real projects come down to one of these:

- **Whole-level trimesh collider**. Passing the world GLB as-is to `ColliderDesc.trimesh` creates a collider with thousands/tens of thousands of triangles. High memory, slow queries, unpredictable contacts. Correct: cuboids/heightfield for the bulk, trimesh only for specific pieces that justify it (a carved rock, a sculpture that *needs* precision).
- **Dynamic bodies where kinematic was enough**. Players and NPCs do not need to be dynamic unless you want physical bounce between them. Kinematic = you move, Rapier detects and optionally gives you correction. Dramatically cheaper and more controllable.
- **Broken sleep**. Rapier automatically sleeps still bodies and they are almost free in that phase. Small forces applied every frame "to make sure", reset transforms, or `setNextKinematicTranslation` even when position has not changed wake them up and lose that advantage.
- **Creating/destroying colliders in the hot path**. Recyclable props, bullets, particles with collision: always pool, activate/deactivate instead of `world.createCollider` + `world.removeCollider` every time.
- **Variable delta in the step**. Passing the `rAF` `dt` directly to `world.step()` instead of using a fixed timestep with accumulator. Produces instability (explosions, tunneling, jitter) and variable cost. Hard rule: the step is fixed; rendering interpolates.
- **Per-frame raycasts per NPC for ground check** when the body's own contact or a targeted sphere cast is enough. Multiplies query cost by N.
- **Sync with per-frame allocations**. `new THREE.Vector3(body.translation().x, ...)` for every body every frame produces visible GC stutter. Use module-level scratch buffers and direct assignment (`obj.position.set(t.x, t.y, t.z)`).
- **Same collider for rendering and simulation**. Already covered in *The collider is never the visual asset*, but with Rapier it hurts especially because the engine lets you do it and does not warn about the cost.

## To expand later
- concrete choice of recommended engine
- patterns for characters and balance
- triggers and sensors
- rollback or reconciliation if there is multiplayer
- debug tools for colliders and contacts
- integration with world generation and streaming
