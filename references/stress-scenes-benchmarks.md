# Stress Scenes and Internal Benchmarks

## Goal
Create stress scenes and internal benchmarks that validate budgets, quality tiers, frame pacing, and architecture decisions in a real Three.js game.

## Main rule
**Do not trust only the pretty prototype scene.**
You need a small battery of scenes that force the project’s typical bottlenecks.

## What a stress scene should answer
- what happens if prop density rises?
- what happens if I enable the high tier?
- what happens if there is loading, spawn, or zone change?
- what happens with high shadows, postprocessing, and pixel ratio?
- what happens on mobile or weaker hardware?

## Useful stress-scene types
### 1. Draw-call stress
Validates:
- instancing vs loose meshes
- geometry merging
- materials per object

Reference pattern:
- baseline `NAIVE`
- variant `MERGED`
- variant `INSTANCED`

The `webgl_instancing_performance` example shows exactly this comparison, and deserves to become an internal reference benchmark.

### 2. Scene graph / CPU stress
Validates:
- cost of updates per frame
- too many live nodes
- per-entity logic
- raycasts or systems that scale badly

Measure:
- frame time
- live nodes
- update frequency
- degradation as entities increase

### 3. Postprocessing stress
Validates:
- composer
- passes
- render targets
- quality tiers

Scale things like:
- bloom on/off
- DOF on/off
- internal resolution of passes
- pixel ratio

This matters a lot because cost here can change brutally between tiers.

When the project uses mirrors, portals, or minimaps, it also deserves a dedicated RTT scene per family. See `render-target-families.md`.

### 4. Asset activation stress
Validates:
- on-demand loading
- `compileAsync()`
- async guards
- stutter when changing model, skin, or scene

Useful test:
- alternate heavy assets or characters
- measure activation spikes
- compare with and without warmup

### 5. Spawn/despawn stress
Validates:
- lifecycle
- pooling vs create/dispose
- budget per frame
- stutter from waves or chunks

### 6. Character/gameplay stress
Validates:
- player controller
- animation
- physics/queries
- camera
- enemy or interaction density

Important because a pretty but empty benchmark can hide the real cost of the game.

## What an internal benchmark should not be
- a demo irrelevant to the real game
- one single “hero shot” scene
- a test with no saved metrics
- a comparison where ten things change at once

## Minimum metrics
Record at least:
- average frame time
- frame-time spikes
- draw calls
- triangles
- geometries/textures/programs
- active tier
- relevant shadow and post configuration

And if applicable:
- build time
- load time
- visible activation time

## Healthy benchmark design
### Control variables
Change one important thing at a time:
- same scene, different prop count
- same content, different tier
- same asset, with and without warmup

If you want to separate CPU and GPU with some honesty, it also helps to have benchmarks where a clean visual lever changes and others where a clean logic lever changes.

### Repeatability
- same seeds if there is randomness
- same camera paths if the view matters
- same activation order
- same quality conditions

### Relevance
Each benchmark should resemble a real threat to the game:
- dense world
- combat with many actors
- scene with heavy post
- chunk change
- 3D selector or inventory

## Recommended minimum set
For a medium project, have at least:
1. **draw-call bench**
2. **postprocessing/tier bench**
3. **asset activation bench**
4. **spawn or chunk bench**
5. **real gameplay slice bench**

## Integration with quality tiers
Benchmarks should be runnable by tier:
- low
- medium
- high

That shows whether the quality system really scales or only changes two cosmetic sliders.

If an adaptive scaler exists, also test:
- how long it takes to react
- whether it thrashes
- whether the downgrade saves frame pacing or only masks the average

## Integration with stutter
Do not look only at the average.
Look at:
- spikes on entry
- spikes when changing tier
- spikes when activating assets
- spikes when recreating composer, shadows, or materials

## Integration with CI or manual reviews
You do not need to automate everything from day 1.
But you should:
- have saved scenes
- be able to open them easily
- know which metrics to look at
- compare before/after large changes

To turn this into more repeatable runs with warmup, measurement window, and structured output, see `benchmark-reporting.md`.

## Expected result
A good benchmark does not say “it is fast”.
It says something like:
- high tier breaks on mobile because of DOF + bloom
- spawning 200 props adds a 40ms spike
- instancing reduces draw calls brutally without breaking the case
- warmup avoids the hitch when changing skin

That is genuinely actionable information.

When those results are saved by run, it is also worth comparing them with a consistent diff layer. See `benchmark-diffs.md`.

## Anti-patterns
- optimizing without a stable test scene
- confusing a synthetic benchmark with the final real experience
- measuring only once
- not saving the test configuration
- not cross-checking render metrics with game events

## Strong recommendation
Create a small `benchScenes` folder or suite, or equivalent, that:
- represents real project threats
- exposes quality and density toggles
- shows basic metrics
- helps compare large changes before calling them good

## To expand later
- reproducible seeds and fixed camera paths
- minimum automation for metric captures
- genre-specific scenarios
