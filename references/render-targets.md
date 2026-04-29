# Render Targets

## Goal
Use render targets in Three.js as real game infrastructure, understanding their cost, lifecycle, and implications for camera, resize, and quality.

## Main rule
**A render target usually means at least one extra render.**
Do not treat it as a free texture or a minor decoration.

## What it really is
A `WebGLRenderTarget` is a texture you render to, but in practice it also implies:
- an additional camera or at least additional configuration
- a scene or scene subset to render
- memory for color and sometimes depth/stencil
- its own resize, cleanup, and quality policy

## Typical use cases
- in-world monitor
- rear-view mirror or security camera
- minimap
- portal or remote view
- color picking in a dedicated target
- auxiliary buffers for custom effects

Shadows and postprocessing also use render targets, but here the focus is their use as an explicit game system.

For concrete families and edge cases, rely on `render-target-families.md`, `portal-recursion.md`, `portal-masking-stencil-scissor.md`, `minimap-fog-of-war.md`, and `fog-mask-blending.md`.

## Recommended default
Before creating a render target, ask:
- do I really need a live view?
- can it update less often?
- can it run at lower resolution?
- can it render a smaller subset of the world?

Very often the correct answer is not “render everything again at full res every frame”.

## Real cost
### 1. Extra render
Each target usually implies another call to `renderer.render(...)`.

That means:
- more CPU submit cost
- more GPU cost
- more culling and traversal work

### 2. Memory
The manual leaves a useful hint:
- by default, `WebGLRenderTarget` creates a color texture and depth/stencil buffer

If you do not need depth or stencil:
- request `depthBuffer: false`
- request `stencilBuffer: false`

That can save useless memory and cost.

### 3. Resize
If the target depends on screen size or viewport:
- `renderTarget.setSize(...)`
- also update the associated camera if the aspect changes

Resizing only the main renderer is not enough.

## Render target camera
The target camera must respond to the target’s real purpose, not blindly copy the main camera.

Examples:
- minimap: probably orthographic
- square monitor: square aspect
- panoramic rear-view mirror: aspect different from the canvas
- picking: camera equivalent to the view where picking happens

## Update frequency
One of the best savings levers.

Not every target needs continuous updates.

Good options:
- every frame, only if critical
- every N frames
- only when something relevant changes
- only while the object is visible
- only when the player interacts with that system

Useful pattern inherited from the responsive and render-on-demand manual:
- if a view does not need continuous motion, do not rerender it nonstop

## Target resolution
Very strong rule:
- tying every target to full canvas resolution is almost never worth it

Think case by case:
- small in-scene monitor, much lower resolution
- minimap, reduced resolution
- blur or auxiliary buffers, half or lower resolution
- UI target or critical capture, maybe high resolution if it adds real value

## Integration with quality tiers
Render targets should be an explicit part of tiers.

Useful variables:
- target on/off
- target size
- update frequency
- depth/stencil on/off when applicable
- content rendered by that target

Do not think only about postprocessing. A CCTV, mirror, or minimap is also a visual budget.

## Integration with adaptive quality
When a target is not critical for gameplay:
- lower its resolution
- lower its update frequency
- turn it off in low tiers

This is often better than touching the main image first.

## Scene subset
The target does not always need to render the entire world.

Healthy patterns:
- layers
- minimal secondary scene
- proxy or simplified content
- hide elements irrelevant to that view

This can be decisive for remote cameras or internal monitors.

In tactical minimaps, it is often worth going further: simplified base map + visibility overlay, instead of rerendering the whole world to express fog of war.

## Basic render order
Typical pattern:
1. configure target
2. `renderer.setRenderTarget(renderTarget)`
3. render associated scene/camera
4. `renderer.setRenderTarget(null)`
5. render main scene

If you mix several targets, keep the order explicit and easy to follow.

## Clear and state
Do not forget the target has its own content and implicit or explicit clear.

Useful questions:
- does it need a full clear every frame?
- does it use its own background?
- does it depend on alpha?
- am I accidentally carrying renderer state over?

In more complex pipelines, order and clear matter quite a lot.

## Feedback loops and weird traps
Avoid situations where a target is used to render something that also accidentally depends on that same target.

Simple rule:
- be careful with mirrors, screens, and surfaces that show the same view currently being generated
- if necessary, temporarily exclude certain objects from the target pass

## Lifecycle
Treat each target as a resource with a clear owner.

Examples:
- minimap system
- security camera system
- diegetic 3D UI screen
- reflector or portal

Each should know:
- when to create
- when to update
- when to resize
- when to `dispose()`

## Cleanup
`scene.remove()` does not clean up the target.

When it is no longer needed:
- `renderTarget.dispose()`
- clean up auxiliary materials or meshes if they were created for that system
- clean up associated cameras, listeners, or passes

## Useful debug
Watch:
- frame time when enabling/disabling the target
- memory and `renderer.info`
- spikes on resize
- number of live targets
- real update frequency

If an “innocent” monitor adds several ms, you have an obvious suspect.

## Anti-patterns
- one full-res target for every little screen in the world
- updating every target every frame by inertia
- forgetting `setSize()` and the target camera’s aspect
- leaving depth/stencil enabled without needing them
- not calling `dispose()`
- rendering the whole world for a view that only needs a subset

## Strong recommendation
Create a small system or wrapper per target that centralizes:
- camera
- resolution
- update frequency
- content filters
- lifecycle and dispose
- integration with quality tiers

## To expand
- picking with a dedicated render target
- multiple targets in HUDs or cockpits
- intermittent update strategies by visibility
