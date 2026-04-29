# GPU vs CPU Bottleneck Heuristics

## Goal
Practically distinguish whether a performance problem in a Three.js game seems to come mostly from GPU, CPU, or another source such as loading/stutter, without pretending impossible precision.

## Main rule
**Do not diagnose by intuition or from a single counter.**
Cross-check signals, run small tests, and compare concrete changes.

## What it tries to solve
- do not lower resolution when the problem is logic or scene graph
- do not remove gameplay when the real problem is postprocessing or shadows
- do not use adaptive quality as a blind patch

## What it does not solve by itself
- deep real GPU profiling
- lifecycle bugs
- stutter from compilation or asset activation
- mixed problems where CPU and GPU step on each other

## The right question
Do not only ask:
- “is it slow?”

Ask:
- does it improve when lowering internal resolution?
- does it improve when removing postprocessing or shadows?
- does it improve when reducing entities, logic, or raycasts?
- is the problem constant, or does it appear during specific events?

## Most useful heuristic: touch one clean lever
### Signs of GPU-bound
Suspect GPU more if it clearly improves when:
- lowering `renderScale` or internal resolution
- lowering pixel ratio or pixel-budget cap
- disabling bloom, DOF, or other expensive passes
- lowering shadows or their resolution
- reducing transparencies or costly materials
- disabling one **specific decor category** (trees, buildings, foliage) even when it has few instances

### “Few assets” does not imply “cheap”

Common trap with AI-generated or scanned assets: a level with **20 trees** can run worse than another with **200**, if those 20 are 1–3M-triangle meshes with `doubleSided`, full PBR, and `receiveShadow`. Instance count lies.

The real GPU cost per asset is roughly:

```
tris × instances × fragments × doubleSided × PBR samples × shadow samples × worldScale²
```

Where `worldScale²` matters because a ×2 scale quadruples the pixels the fragment shader has to process. Any one of those multipliers set “by accident” kills the budget.

Symptoms that point here:
- disabling an entire decor category (trees, buildings) with a flag gains 15–30 fps even though the category only has 20 instances
- `renderer.info.render.triangles` in the tens of millions in an “empty” scene
- the adaptive scaler lowers resolution but frame time barely changes

Fix: go to the GLB and lower triangles/`doubleSided`/texture, not the renderer. `InstancedMesh` **collapses draw calls but does not lower per-instance cost**: if the mesh is obese, the InstancedMesh is too.

### Signs of CPU-bound
Suspect CPU more if it clearly improves when:
- reducing entities with per-frame updates
- lowering frequency of secondary systems
- removing expensive raycasts or queries
- simplifying gameplay, AI, or physics logic
- reducing live nodes or scene graph work
- avoiding geometry rebuilds or mass JS changes

### Signs of load-bound or stutter-bound
Suspect another category if the problem appears mainly when:
- entering a scene
- changing skin or model
- enabling materials or passes
- spawning or destroying many things at once
- changing tier live

There, `frame-pacing-stutter.md` usually matters more than a simple CPU/GPU dichotomy.

## Quick tests that often say a lot
### 1. Resolution test
Lower internal resolution visibly.

If the frame improves a lot:
- smells like GPU

If it barely changes:
- GPU probably was not the main bottleneck

## 2. Postprocessing test
Disable composer or expensive passes.

If the frame improves a lot:
- strong GPU/postprocess suspect

If it barely changes:
- look elsewhere

## 3. Logic density test
Reduce active entities, updates, raycasts, or systems.

If it clearly improves:
- strong CPU suspect

## 4. Draw-call test
Reduce loose meshes, distinct materials, or visible density.

If `renderer.info.render.calls` drops and performance improves a lot:
- it may be a CPU bottleneck from driver/submit and also GPU from draw overhead
- do not assume draw calls are “GPU-only”

## 5. Scene graph test
Replace a mass of meshes with instancing or merge in an equivalent scene.

If it improves:
- there was important cost in node maintenance, draw calls, or both

## `renderer.info` as aid, not supreme judge
Look at:
- `render.calls`
- `triangles`
- `geometries`
- `textures`
- `programs`

Useful readings:
- very high `render.calls`, suspect draw overhead
- `programs` growing when touching materials, suspect permutations and compilation
- geometries/textures growing without dropping, suspect lifecycle or leak

But `renderer.info` alone does not tell you “this is CPU” or “this is GPU”.

## Scene graph and CPU
The optimization manuals leave a very strong clue:
- too many nodes, helpers, and transforms also cost even when render does not look outrageous

Suspect CPU when:
- the scene has thousands of objects with updates
- the problem drops when reducing logical entities
- instancing/merge improves things without changing shading much

## Material changes and false diagnoses
`how-to-update-things` makes clear that certain material changes force recompilation.

That can look like “slow GPU” when you are actually seeing:
- shader compilation
- stutter from `needsUpdate`
- costly reconfiguration at a bad time

Rule:
- separate slow throughput from recompilation spikes

## Resolution and pixel budget
The `responsive` manual leaves another strong pattern:
- explicitly controlling the drawing buffer helps make clean GPU tests
- capping pixel count avoids drowning in HD-DPI

That makes the resolution test fairly reliable as a first practical filter.

## Quality scaling and diagnosis
The adaptive scaler should not act only on raw frame time.

If there are signs of CPU-bound:
- lowering resolution may help little
- it may even hide the real problem

Healthy pattern:
- use lightweight heuristics to estimate whether the cost looks more visual or more logical
- prioritize `renderScale` and post cuts when it smells like GPU
- do not touch visual quality aggressively if symptoms point to CPU or activation stutter

## Recommended benchmarks for separating causes
It is useful to have distinct benchmarks for:
- fill/postprocessing heavy
- draw-call heavy
- scene graph/update heavy
- spawn/activation heavy

If everything goes into one giant scene, diagnosis turns into mud.

And if you also want to compare runs over time, those benchmarks should emit consistent reports. See `benchmark-reporting.md`.

## Mixed cases
Very common:
- high draw calls hit CPU and GPU
- shadows hit GPU, but can also trigger variants and extra work
- too many objects hit scene graph and render submission too

Rule:
- accept that sometimes the main bottleneck changes by device or scene
- look for the most profitable lever, not a perfect label

## Pocket heuristic
### Smells like GPU if
- lowering resolution helps a lot
- removing post/shadows helps a lot
- logic barely changes the result

### Smells like CPU if
- lowering resolution helps little
- reducing entities/updates helps a lot
- devtools show heavy JS or costly callbacks

### Smells like stutter/load if
- the average does not look terrible but there are concrete hits
- the problem appears when enabling, loading, compiling, or reconfiguring

## Anti-patterns
- assuming low FPS on mobile automatically means GPU
- lowering visual quality before isolating CPU
- blaming triangles when the problem is draw calls or logic
- blaming draw calls when the problem is shader compilation
- trying to solve activation stutter with a simple visual downgrade

## Strong recommendation
Have a short diagnostic routine:
1. measure frame time and context
2. run the resolution test
3. run the post/shadows test
4. run the entities/update test
5. cross-check with `renderer.info` and devtools
6. provisionally label: GPU, CPU, mixed, or stutter/load

## To expand later
- heuristics by game genre
- finer signals with browser tooling
- diagnosis by device or tier
- direct integration with `adaptiveScaler`
