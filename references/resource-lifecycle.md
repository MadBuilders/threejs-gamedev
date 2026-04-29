# Resource Lifecycle

## Goal
Avoid leaks, memory spikes, and silent degradation in Three.js games by managing the real lifecycle of geometries, materials, textures, render targets, and auxiliary objects.

## Main rule
Removing something from the scene **does not** mean its resources are freed.

The manual states it plainly: if you no longer need a geometry, material, texture, or render target, you usually have to call its `dispose()` explicitly.

## What to clean up

### Geometries
- `BufferGeometry.dispose()`

### Materials
- `Material.dispose()`

### Textures
- `Texture.dispose()`
- if there is an `ImageBitmap`, also close the bitmap when applicable

### Render targets
- `WebGLRenderTarget.dispose()`

### Skeletons
- `Skeleton.dispose()` if it is no longer shared with other skinned meshes

### Addons and utilities
Many addons also have `dispose()`:
- controls
- postprocessing passes/composer
- utilities with listeners or internal buffers

## Ownership rule
Every resource should have a clear owner.

Examples:
- a world chunk owns its aggregated mesh and temporary textures
- a UI scene owns its render targets
- a postprocessing system owns its composer and passes

If nobody knows who cleans something up, it will probably stay alive longer than it should.

## When to clean up
Typical good moments:
- level change
- chunk unload
- leaving a scene or game mode
- massive replacement of assets or representation strategy

In pipelines with glTF loaded on demand, also add:
- quick switching between models or skins
- asynchronous loads that become obsolete
- viewers or selectors where the user can change assets before loading finishes

## Shared resources
Do not destroy shared resources recklessly.

Before calling `dispose()`, ask:
- does another material use this texture?
- is this skeleton shared?
- do more meshes use this material?

The animation examples reinforce this strongly: cloning characters with `SkeletonUtils.clone()` and sharing a skeleton are two different strategies, so cleanup changes too.

## Renderer info
Use `renderer.info` to watch:
- geometries
- textures
- programs
- draw calls and frame stats

It is not the absolute truth for the entire system, but it is a very useful alarm for leaks or odd growth.

## Reusing after dispose
The manual clarifies something useful:
- in many cases Three.js can recreate resources if you use the object again after `dispose()`
- that does not always break runtime, but it can add cost to the frame

In other words, misused `dispose()` does not always explode. Sometimes it just hurts performance.

## Recommended pattern
1. decouple logical data from GPU resources
2. register what each subsystem creates
3. have an explicit cleanup path
4. clean up by coherent groups, not with a hundred loose patches

## Anti-patterns
- assuming `scene.remove()` is enough
- not cleaning render targets or composer
- not checking `dispose()` in addons
- destroying shared resources without clear ownership
- not looking at `renderer.info` when you suspect a leak

## Strong recommendation
In games with chunks, scenes, or modes:
- every large unit of the system should have `create`, `attach`, `detach`, `dispose`, or equivalent
- explicit lifecycle always beats implicit magic

For ownership, resize, and update policy of custom RTTs, see `render-targets.md`.

## To expand later
- cleanup checklist by scene/chunk
- pooling vs dispose
- lifecycle of composers and passes
- asset streaming and deferred discard
- relationship between cleanup and frame stutter
