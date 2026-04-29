# Profiling and Budgets

## Goal
Have a practical way to measure performance in Three.js games, identify real bottlenecks, and work with budgets instead of vague intuition.

## Main rule
**Do not optimize blind.**
First distinguish what is failing:
- initial load
- CPU per frame
- GPU per frame
- memory
- stutter from compilation or resource creation

## The right question
Do not only ask “is it slow?”.
Ask:
- when is it slow?
- in which scene or system?
- is it constant or spiky?
- is it dropping because of draw calls, logic, shaders, textures, or async loads?

## Budget before panic
Design with budgets from the start.

Typical budgets to watch:
- target frame time
- draw calls
- live geometries and nodes
- texture memory
- number of expensive lights
- number of objects updated per frame
- scene or zone load time

You do not need to pretend there are magic universal numbers, but you do need to choose concrete limits per project and revisit them.

## Frame time rule
Think in milliseconds per frame, not only FPS.

Useful reference:
- ~16.7ms for 60fps
- ~33.3ms for 30fps

If a new system adds 4, 6, or 8ms by itself, you already know who is asking for too much.

## Useful problem separation
### CPU-bound
Suspect:
- too many updates per frame
- heavy gameplay logic
- excessive raycasts
- too many scene graph nodes
- frequent merges or rebuilds
- avoidable JS work
- **per-frame allocations in the hot path**: `vec.clone()`, `array.map(...)`, `const arr: Foo[] = []` inside the loop. Individually they are cheap; multiplied by 60 Hz and every entity, they generate GC pressure that appears as rhythmic stutter, not constant low FPS. Healthy pattern: reusable **module-level scratch buffers** (preallocate a `Vector3` and use it with `.copy()`/`.set()`; persistent array with `length = 0` at the start of the frame and `push()` only when it grows). Important: if scratch is aliased in a public `result`, clear it in-place in `reset()` instead of reassigning (`arr.length = 0`, not `arr = []`), or callers keep seeing old data.

### GPU-bound
Suspect:
- high draw calls
- too many shadows
- expensive postprocessing
- too many transparencies
- heavy materials
- resolution or pixel ratio too high

### Load-bound or stutter-bound
Suspect:
- assets that are too heavy
- costly decompression or parsing
- shader compilation
- mass creation or destruction of resources at a bad time

## Minimum tools
### `Stats`
Useful for seeing whether the frame degrades in an obvious and quick way.
It does not explain everything, but it helps notice drops and compare changes.

### `renderer.info`
Look especially at:
- geometries
- textures
- programs
- render.calls
- triangles

This does not tell the whole story, but it gives a very good read on render state.

### Local measurement
Useful simple patterns:
- `console.time()` / `console.timeEnd()` for builds or expensive steps
- measure load duration
- measure procedural generation
- measure geometry creation or rebuilds

The `webgl_instancing_performance` example uses this approach quite honestly and is worth keeping as a pattern.

### Browser devtools
Use browser profiling when the problem is unclear.
Especially for:
- JS flame charts
- memory
- layout or external UI spikes
- callback and listener cost

## `renderer.info` as compass
Useful pattern:
- if `render.calls` spikes, think draw calls
- if geometries/textures grow without dropping again, think lifecycle or leak
- if triangles rise a lot in scenes where they should not, review assets and LOD

Do not obsess over a single counter. Cross-check it with scene context.

## Draw calls
The manual and examples send a very clear signal:
- **many draw calls kill sooner than many people think**

Typical solutions:
- `InstancedMesh`
- geometry merging
- fewer distinct materials
- fewer useless tiny meshes

Important tradeoff:
- instancing reduces draw calls while keeping some flexibility
- merge reduces draw calls but complicates individual updates
- naive meshes are fine for prototypes, not as the default for large quantities

## Scene graph and CPU cost
The docs also leave another unglamorous truth:
- drawing is not the only cost; maintaining thousands of nodes, matrices, and updates also costs

Rule:
- if you are only representing mass data or repeated props, do not use the scene graph as a gigantic data structure just because you can

## Expensive updates
`how-to-update-things` leaves several important warnings:
- changing certain material properties can force recompilation
- resizing buffers is not cheap
- dynamic geometries need prealloc and well-planned updates
- changing instancing or skinned bounds may require recomputing bounding volumes

Implication:
- do not measure only final render; also measure the cost of mutating data and resources

## Recommended budget categories
Not as dogma, but as discipline.

### Render
- guideline draw-call limit per playable scene
- limit for shadow-casting lights
- limit for postprocessing passes

### Assets
- maximum size per critical model
- maximum texture size by category
- number of materials per important asset

### Gameplay
- how many entities update fully every frame
- how many raycasts or queries are allowed per tick
- how many systems can run at lower frequency

### World
- maximum density per chunk
- simultaneous visible props
- spawn/despawn budget per frame

## Budgets by tier
Especially on web and mobile, define tiers:
- low
- medium
- high

Candidate variables:
- pixel ratio
- shadows
- prop density
- draw distance
- postprocessing
- texture quality or asset variants

## Healthy profiling pattern
1. reproduce the problem
2. isolate the scene or system
3. measure before the change
4. apply one clear intervention
5. measure after
6. write down the tradeoff

For a more explicit practical diagnosis routine between GPU, CPU, mixed, or stutter/load bottlenecks, see `gpu-vs-cpu-heuristics.md`.

## What to measure periodically
- average frame time and spikes
- draw calls in key scenes
- approximate live memory
- load times per scene
- procedural generation cost
- stutter when changing zone, skin, or model

For more concrete doctrine on stutter, warmup, and smooth activation, see `frame-pacing-stutter.md`.

## Anti-patterns
- optimizing by superstition
- looking only at FPS without frame time
- fixing GPU by lowering quality when the bottleneck is CPU
- fixing CPU by removing geometry when the problem is shaders or shadows
- assuming a synthetic benchmark represents your real game
- not measuring load spikes and only looking at the average

## Strong recommendation
Create a small `performanceHUD` or debug panel that can show:
- fps or frame time
- `renderer.info.render.calls`
- geometries/textures/programs
- active quality tier
- toggles for shadows, post, and density

That usually pays for itself as soon as the project grows a little.

To verify all this against reproducible scenes and not just intuition, see `stress-scenes-benchmarks.md`.

To save comparable runs with context, warmup, and structured results, see `benchmark-reporting.md`.

To compare baseline vs candidate without falling into misleading diffs from different config or noise, see `benchmark-diffs.md`.

## To expand later
- finer GPU timing by browser and tooling
- guideline budgets by genre
- automated stress-scene tests
- integration with adaptive quality scaler
