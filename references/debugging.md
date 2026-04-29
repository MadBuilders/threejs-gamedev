# Debugging

## Goal
Have a practical debugging strategy for Three.js games that lets you quickly detect camera, transform, light, shadow, material, asset, and general frame-flow problems.

## Main rule
Make the game and scene state visible. If something fails and cannot be inspected, time goes to hell.

## Useful helpers
Use visual helpers when they add clarity:
- `AxesHelper`
- `GridHelper`
- `BoxHelper`
- `Box3Helper`
- `CameraHelper`
- `SkeletonHelper` when there are characters or rigs
- light helpers when relevant
- custom gizmos or markers if the game needs them

## What should be inspectable
- position, rotation, and scale of key objects
- node hierarchy loaded from assets
- active camera and its real target
- colliders or approximate volumes
- spawn points
- interaction zones
- state of important animations
- active base action, weights, and additive layers if they exist
- time, delta, and update order
- resource ownership if a scene loads and unloads assets
- `renderer.info` when you suspect a leak or odd growth

## Checklist of common problems

### Nothing is visible
- camera badly placed
- absurd near/far values
- object outside the frame
- broken scale
- renderer not mounted correctly
- canvas with incorrect size
- insufficient lights if the material needs them
- material or texture misconfigured

### The model loads oddly
- incorrect axis or orientation
- strange pivot
- inconsistent scale
- broken materials
- missing textures
- dirty or unexpected hierarchy

### Shadows are terrible
- too many active shadows
- expensive shadow map for the real scene
- shadow camera poorly adjusted
- objects marked for cast/receive shadow without discipline
- expecting perfect shadows on cheap mobile devices

### Performance drops
- high draw calls
- too many expensive lights
- excessive geometries or textures
- unnecessary postprocessing
- too many objects updating every frame
- uncontrolled raycasts or calculations scattered across systems

## Useful strategies
- introduce debug toggles from the beginning
- be able to enable and disable helpers quickly
- add a debug panel if the project grows
- have a minimum performance panel with frame time/FPS, `renderer.info`, and quality tier
- log useful warnings when loading assets
- isolate systems to test whether the problem is in input, update, or render
- distinguish whether the failure comes from the scene graph, asset pipeline, or resource lifecycle

## Recommended minimum visual debug
- show axes or grid in prototypes
- be able to draw bounds for important entities
- highlight selected or interactive objects
- visualize control points, triggers, and spawns
- visualize hit points, normals, or raycast markers when there is 3D interaction

## Healthy rules
- do not debug only by reading code
- do not accidentally leave permanent helpers in production
- do not assume the problem is in Three.js before checking camera, scale, and state
- debug the basic and cheap things first
- also check input focus, resize, and resource release when the problem feels “random”

## Anti-patterns
- twenty unstructured `console.log`s
- helpers scattered through the code without control
- mixing debug tools with final gameplay logic
- no way to inspect the scene graph or asset hierarchy
- optimizing blindly without identifying the bottleneck

## To expand later
- stats and per-frame metrics
- material inspection tools
- animation and mixer debugging
- loader and streaming debugging
- mobile-specific checklist
