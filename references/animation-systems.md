# Animation Systems

## Objective
Use the Three.js animation system as a serious game subsystem, not as a series of loose `play()` calls glued to the loader.

## Main rule
Clearly separate:
1. **assets and clips**
2. **mixer and actions**
3. **animation state**
4. **transition rules**
5. **synchronization with gameplay**

## Base pieces of the system
The official system revolves around:
- `AnimationClip`
- `KeyframeTrack`
- `AnimationMixer`
- `AnimationAction`
- optionally `AnimationObjectGroup`

## Recommended default
- one `AnimationMixer` per character or animated root, except in special cases
- treat each clip as data
- treat each `AnimationAction` as runtime control
- keep an animation module or system separate from input and game rules

## Healthy pattern
Do not do this:
- load glTF
- create mixer
- call `clipAction(...).play()` anywhere
- cross your fingers

Do this:
- register clips by name
- create controlled actions
- define base state (`idle`, `walk`, `run`, etc.)
- encapsulate transitions and weights
- update the mixer in the loop with `delta`

## Base actions vs additive actions
The official examples leave a very useful separation:

### Base actions
Main states that are mutually exclusive or almost:
- idle
- walk
- run
- jump loop

### Additive actions
Partial layers or additional poses:
- head shake
- aim
- upper-body pose
- sneak pose
- gesture or reaction

Strong rule:
- do not mix both categories without naming them
- base actions drive the main body
- additive actions adjust on top with controlled weights

## Crossfades
The official system supports crossfade, but that does not mean every transition should be fired without criteria.

Recommended pattern:
- centralize transitions
- use short, consistent durations
- reset time and weight when appropriate
- if a transition depends on the end of the current loop, synchronize it explicitly

## Time scale and weights
`AnimationAction` and `AnimationMixer` can change:
- weight
- speed
- paused state
- repetition

That is powerful, but also easy to turn into chaos.

Rule:
- gameplay decides intent
- the animation system decides effective weights, crossfades, and timeScale

## Update loop
Mandatory rule:
- update `mixer.update(delta)` in the main loop
- use the real frame `delta`
- do not depend on hardcoded times outside the system

## Locomotion and state machine
In games with a controllable character, animation should hang from a locomotion state that is more stable than raw keyboard input.

Recommended pattern:
- input -> locomotion intent
- locomotion/controller -> character state
- animation system -> clip, blending, and layer resolution

Useful states for animation:
- idle
- move
- sprint
- jumpStart
- airborne
- land

Strong rule:
- do not trigger `walk` because `W` is pressed
- trigger `walk` or `run` because the character is actually moving according to its locomotion state

For explicit design of states, priorities, one-shots, and conceptual layering, see `animation-state-machines.md`.

## Root motion
Decide early whether actual movement comes from gameplay/controller or from the animated clip.

Initial recommendation:
- gameplay-governed locomotion by default
- root motion only in specific cases where it truly pays off

Reason:
- simplifies collisions
- simplifies multiplayer
- simplifies synchronization between camera, controller, and animation

## Character cloning
The examples teach something very useful:
- use `SkeletonUtils.clone()` to duplicate animated characters
- distinguish between clones with independent skeletons and setups with a shared skeleton

Recommendation:
- full independence if each character can have different state
- shared skeleton only if you really want to share state and know what you are doing

### Concrete gotchas when cloning SkinnedMesh
Proven in production when cloning one character for multiple actors (NPCs, remote players, etc.). These errors are silent until they happen, and when they happen they cost time:

- **`Object3D.clone()` is not enough for skinned meshes**. The native clone shares the `Skeleton` by reference and all actors end up with the same pose (or NaN). Use `clone` from `three/examples/jsm/utils/SkeletonUtils.js`. Typical symptom: "all clones move the same", or "the clone appears in T-pose".
- **Exports from `SkeletonUtils.js` are named, not a namespace**. Import as `import { clone as cloneSkinned } from 'three/examples/jsm/utils/SkeletonUtils.js'`. `import { SkeletonUtils } from '...'` fails at runtime with `does not provide an export named 'SkeletonUtils'`.
- **`SkeletonUtils.clone` shares materials by reference**. If you tint one clone, you tint them all. For per-instance tints, traverse the clone and call `material.clone()` per mesh (then multiply `.color` by the tint). Multiplicative tinting on `.color` preserves the underlying texture; replacing the material breaks the look.
- **`SkeletonUtils.clone` also shares geometries**, which is good (one GPU upload for all). But it means **disposing geometry from one clone's `dispose()` breaks all the others**. Rule: the clone disposes only what it cloned (tinted materials, mixer, sprite tags). Geometry lives with the source.

### “source + instance” pattern for shared assets
When the same GLB is used in N actors (local player + NPCs, local player + remotes, etc.), separate asset loading from instance creation:

- `loadCharacterSource(url)`: fetch + parse the GLB with cache by URL. Returns an immutable object with `{ scene, clips }`.
- `createCharacterInstance(source, opts)`: synchronous. Runs `cloneSkinned(source.scene)`, sets up its own `AnimationMixer`, optionally clones and tints materials, and returns the runtime API (`tick`, `dispose`, `getJugWorldPosition`, etc.).
- `loadCharacter(url, opts)`: async helper for simple call sites. Internally `await loadCharacterSource(url)` + `createCharacterInstance(source, opts)`.

Advantages:
- One download + parse per URL.
- Spawning the second actor is a sync `clone` + setup, with no network wait.
- The local player call site does not change.
- The “remote actors” system (NPC manager, remote player, etc.) can instantiate inside a callback without awaits.

Apply the same split to non-animated props (e.g. `loadJugSource` + `createJugInstance`): plain `Object3D.clone(true)` is enough because there is no skeleton, but the cache + clone-materials-only-if-tinting pattern keeps the rules clear.

### Clone animation
Each clone needs its own `AnimationMixer` running on its own cloned scene. `AnimationClip` instances are pure data, so the clone's `mixer` can reuse the clips parsed in the source safely (clean scale tracks **once** in the source, not per instance).

## Skeletons and cleanup
If a skinned mesh or skeleton is no longer used:
- check whether the skeleton is shared
- if it is not, clean up with `Skeleton.dispose()` when appropriate
- in the source/instance pattern: the source lives as long as the page; the instance `dispose()` only cleans up mixer + cloned materials + its own sprites. Original geometries and skeletons are left alone.

## Bounding volumes
The official update guide reminds us of something important:
- `SkinnedMesh` has its own bounding volumes
- if the game depends on reliable culling, queries, or debug over an animated mesh, review bounding boxes/spheres in relevant states

## Useful debug
- `SkeletonHelper`
- panel for weights and active actions
- visualization of current state (`idle`, `walk`, etc.)
- toggles for additive layers
- machine state and active transition if one exists

## Anti-patterns
- animation mixed directly with input
- `clipAction(...).play()` scattered through the codebase
- not distinguishing base vs additive
- not centralizing crossfades
- not naming clips and depending on magic indices
- cloning animated characters without thinking about skeletons and ownership
- using `Object3D.clone()` on SkinnedMesh and then chasing why clones move identically
- tinting the original material while expecting clone independence
- disposing geometry in a clone's `dispose()` (breaks all the others)

## Strong recommendation
For real games, create an `animationSystem` or `characterAnimationController` that:
- indexes clips by name
- exposes high-level intents
- resolves transitions
- updates the mixer
- knows how to clean resources if the actor disappears

If the character already has multiple actions, combat, or partial layers, also separate out an explicit `characterAnimationStateMachine`.

## To expand later
- root motion vs gameplay-governed movement
- upper/lower body layering
- integration with combat or advanced locomotion
- multiplayer and animation state replication
