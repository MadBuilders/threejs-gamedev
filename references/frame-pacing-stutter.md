# Frame Pacing and Stutter Control

## Goal
Reduce visible hitches and irregular frame pacing in Three.js games, understanding that a good average FPS does not guarantee good game feel.

## Main rule
**Do not measure only average FPS.**
Watch:
- frame-time spikes
- stutter when loading or activating assets
- shader recompilation
- mass work concentrated in one frame

## What actually hurts
Typical feel problems:
- entering a zone and noticing a micro-freeze
- changing skin or character and breaking the frame
- enabling a pass or material and getting a hitch
- spawning a mass of props or enemies and feeling a hard hit

That usually comes from work concentrated at a bad time, not only from low average performance.

## Types of stutter
### Shader stutter
- shader compilation or recompilation
- material or flag changes that force new programs

### Asset activation stutter
- parsing
- decompression
- uploading resources to GPU
- first use of materials or textures

### JS/update stutter
- large procedural generation
- geometry rebuilds
- mass spawn/despawn
- too much logic grouped into one frame

### Postprocessing stutter
- enabling composer or expensive passes live
- poorly coordinated resize
- creating render targets at a bad time

## `compileAsync()` as a serious pattern
The modern `webgl_loader_gltf` example leaves a very valuable pattern:
- using `renderer.compileAsync()` before adding an important model can avoid the compilation hitch on the first visible frame

Cases where it is worth considering:
- character or skin changes
- entering a new scene
- model viewer or selector
- large assets loaded on demand

## Avoidable recompilation
`how-to-update-things` gives a fairly clear warning:
- certain material properties do not change for free
- some force recompilation and can induce jerkiness

Be especially suspicious of changes that alter:
- presence of textures
- vertex colors
- morphing
- shadow map usage
- alpha test
- transparent
- uniform structure or shader variants

Rule:
- do not casually change material permutations in the middle of gameplay
- prefer dummy values or prepared paths if the change will be frequent

## Mental prewarm
Think in terms of “prewarm” or early preparation of resources.

Examples:
- compile models or materials before they enter view
- load and prepare an enemy before visible spawn
- build render targets or passes before an important transition
- precompute variants you know you will need soon

## Staggering
Do not dump heavy work all at once if it can be spread out.

Useful patterns:
- create props in batches
- spread spawn over several frames
- build chunks by phases
- stagger initialization of secondary systems
- do not clear and rebuild half the world in the same frame if there is an alternative

## Asset activation
Loading an asset does not end when the file arrives.
Sometimes it still needs:
- parse
- texture binding
- material compilation
- animation activation
- camera adjustment or scene attachment

Strong rule:
- separate **load complete** from **asset ready to show smoothly**

## Frame time and spikes
More useful than looking only at FPS:
- measure average frame time
- measure clear spikes
- inspect what happens at specific events: spawn, load, resize, preset change, scene entry

If the game runs reasonably well but “feels bad”, this layer is almost always where you need to look.

## Delta and stability
If delta blows up because of a pause, load, or tab switch:
- do not let gameplay explode by integrating a monstrous delta
- clamp maximum deltas in sensitive systems when it makes sense
- use substeps in systems that need them

This does not fix the whole cause of stutter, but it prevents a bad frame from destroying the simulation.

## Spawn, despawn, and lifecycle
Dangerous moments:
- enemy waves
- chunk change
- opening a 3D inventory or selector
- changing visual quality
- destroying and recreating composer or materials

Recommendation:
- clear ownership
- explicit create/attach/detach/dispose
- avoid spikes from unplanned mass destruction and creation

## Postprocessing
Passes can introduce hitches too, not just continuous cost.

Be careful with:
- enabling bloom or new chains live
- resize without synchronizing renderer and composer
- creating large render targets during critical gameplay

Quality tiers help a lot here if they explicitly control when and how passes, sizes, and targets change.

## Useful debug
Have a panel or logs that can correlate a hitch with an event:
- current frame time and recent spikes
- asset loading or activation
- quality changes
- number of programs or draw calls
- important spawn/despawn moment

## Anti-patterns
- bragging about average FPS while ignoring spikes
- changing material flags live without thinking about recompilation
- loading and showing a large asset at the same critical instant
- massive geometry rebuild in a playable frame
- enabling heavy postprocessing without warmup
- not separating “loaded” from “ready to show smoothly”

## Strong recommendation
Create a small runtime policy covering:
- preload
- compile/warmup
- visible activation
- staggering heavy work
- delta limits in sensitive systems
- spike measurement, not only average measurement

If the project also uses automatic quality scaling, that policy should include hysteresis, cooldown, and changes at safe moments. See `adaptive-quality-scaling.md`.

## To expand later
- warmup by scene or encounter
- more concrete background-preparation techniques
- quality scaler based on frame-time spikes
- spawn budget policy per frame
