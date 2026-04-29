# Mobile and Performance

## Goal
Build Three.js games that remain viable on mobile web and modest devices, while avoiding visual decisions that kill the experience too early.

## Main rule
Design with a **performance budget** from the start.

Do not think about performance only once the game is already struggling. Every new system should pay rent.

## Correct priority
On mobile, these usually matter more than raw detail:
- stable frame rate
- reasonable load times
- responsive controls
- good visual readability
- battery and temperature under control

For a more explicit measurement and budget methodology, see `profiling-budgets.md`.

## Base principles

### 1. Lower cost, better frame
Prefer:
- fewer draw calls
- fewer expensive lights
- fewer dynamic shadows
- fewer problematic transparencies
- less useless geometry
- less default postprocessing

### 2. Scale by tiers
Think in qualities or presets:
- low
- medium
- high

Typical variables to trim:
- pixel ratio
- shadows
- draw distance
- prop count
- post effects
- texture resolution

For a more explicit preset policy and coordination between renderer, composer, and render targets, see `quality-tiers.md`.

### 3. Measure before guessing
Do not optimize blind.

Separate the questions:
- is the bottleneck in GPU?
- is the bottleneck in CPU?
- is the problem initial loading?
- is the problem too much per-frame logic?

## Healthy mobile defaults
- cap `renderer.setPixelRatio()` to reasonable values
- avoid complex dynamic shadows by default
- use a few important lights
- prefer simple backgrounds or cheap skyboxes over very expensive environments
- use geometries and materials that match the game’s real style
- start with fewer effects and raise quality only if there is margin

In more serious projects, it is worth going one step further and explicitly controlling drawing-buffer size and pixel budget per device, not just using `setPixelRatio()` casually.

## Draw calls
Draw calls are often one of the first ceilings.

Reduce them with:
- `InstancedMesh` when there are many similar objects
- geometry merging if it makes sense
- fewer distinct materials
- fewer useless decorative objects

The manual and examples reinforce that `InstancedMesh` is not a weird trick, but a central tool when the world needs many similar objects.

Useful tradeoff seen in examples:
- `InstancedMesh` is a very strong solution when quantity and cost dominate
- geometry merging also reduces draw calls, but sacrifices flexibility for individual updates
- many loose meshes should be the exception, not the default, for repeated props

## Geometry and meshes
- watch real polycount, not just appearance
- avoid hyper-dense assets if they will appear small on screen
- review LOD or simplified variants if the world grows
- do not assume a beautiful desktop asset works equally well on mobile

Another lesson from the optimization manual: do not use the scene graph as a massive data structure if what you need is to represent thousands of elements. The cost is not only drawing; it is also maintaining too many live nodes.

## Materials and lights
- start simple
- justify every expensive light
- use `MeshStandardMaterial` or similar thoughtfully, not by inertia
- check whether some objects can use cheaper materials
- treat shadows as a controlled luxury, not a universal right

## Textures
- reasonable sizes
- avoid 4K for posturing
- reuse textures when possible
- compress if the pipeline allows it
- watch total memory, not just disk size

## Transparencies and postprocessing
Both can become expensive and troublesome.

Use them deliberately:
- transparencies only when they add something real
- modular, toggleable postprocessing
- do not chain effects just because you can

## Update loop and CPU
Not every performance problem is in rendering.

Review:
- how many objects update per frame
- how many raycasts you do
- how much logic runs even when nothing changes
- how many unnecessary listeners or sync points exist
- whether some systems could run at a lower frequency

## Practical strategies
- visual budget from the prototype
- quality presets
- toggles for shadows, effects, and world density
- periodic profiling, not only at the end
- real mobile-device tests as early as possible

Useful pattern from the manual: in screens or views that do not need constant updates, consider render on demand. A main game usually has a continuous loop, but 3D menus, configurators, or paused scenes do not need to render nonstop.

## Quick checklist when something goes wrong
- pixel ratio too high?
- too many draw calls?
- excessive shadows?
- textures too large?
- unnecessary postprocessing?
- too many objects updating every frame?
- raycasts or collisions too frequent?
- does the problem appear during loading, gameplay, or specific scenes?

And, very importantly, do not assume every mobile problem is GPU: separate visual vs logic with `gpu-vs-cpu-heuristics.md`.

## Anti-patterns
- designing for powerful desktop and expecting miracles on mobile
- enabling shadows and premium effects from day 1
- adding heavy generative assets without pruning
- using expensive materials everywhere
- trying to fix FPS only by lowering visual quality without checking CPU
- not testing on real devices until the end
- ignoring official instancing techniques and continuing to push thousands of loose meshes

## To expand later
- guideline budget by game type
- LOD and chunking strategies
- practical limits for shadows
- profiling with concrete tools
- adaptive quality policy by device
