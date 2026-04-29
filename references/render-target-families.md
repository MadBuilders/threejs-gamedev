# Render Target Families: Mirrors, Portals, Minimap

## Goal
Ground three very common RTT families in Three.js games, understanding what makes them different, what traps they bring, and which defaults are usually healthy.

## Main rule
**Not all render targets behave the same.**
A mirror, a portal, and a minimap share infrastructure, but they do not share the same camera, the same update frequency, or the same acceptable cost.

## 1. Mirrors

### What they really are
A planar mirror is not just “another camera looking at the same thing”.
It needs:
- camera reflected relative to the plane
- correct clipping to avoid seeing things behind the mirror
- hiding the mirror itself during its render

The `Reflector` addon makes this pattern very clear.

### Useful pattern seen in `Reflector`
- creates a `WebGLRenderTarget`
- generates a reflected camera for each scene camera that uses it
- updates a texture matrix
- modifies the projection with an oblique clip plane
- hides the reflector during the pass
- restores the previous render target when finished

That is no longer a “little monitor”. It is fairly serious infrastructure.

### Healthy defaults
- use `Reflector` if the case is a standard planar mirror
- lower `textureWidth/textureHeight` before degrading the whole scene
- do not casually use full-res mirrors on mobile
- update target size on resize

### Costs and risks
- expensive extra pass
- multiple mirrors multiply cost very quickly
- risk of visual feedback or recursion if the mirror sees other mirrors
- resize spikes if the target follows the main drawing buffer

### When to disable or trim
- low tiers
- secondary or decorative mirrors
- scenes with several simultaneous reflectors

Good levers:
- target resolution
- update frequency
- disable distant or invisible mirrors

## 1.5 Refractors

### What they really are
A planar refractor shares a lot of infrastructure with a planar mirror:
- render target
- oblique clip plane
- hiding the surface itself during the pass

But it does not sell a specular reflection of the world. It sells a refracted or distorted view through a surface.

The `Refractor` class and the `webgl_refraction` example make this quite clear.

### What changes compared with a mirror
- uses a virtual camera copied from the main camera instead of a reflected camera
- the final result depends much more on the refraction shader
- often relies on auxiliary maps, such as dudv, for distortion
- visually it can tolerate more modest resolution if the shader does its job well

### Healthy defaults
- treat it as a premium surface, not free decoration repeated everywhere in the level
- lower `textureWidth/textureHeight` before touching the whole scene
- measure whether it truly adds more than a fake solution or cheaper material
- use continuous updates only if the surface truly needs them

### Typical risks
- confusing it with a mirror and expecting the same geometric credibility
- raising resolution a lot to hide a weak shader
- forgetting that distortion can also hurt readability
- adding too much refractive water/glass and wasting GPU for no reason

### When it is worth it
- hero water or glass
- specific magic or sci-fi surfaces
- moments where distortion adds real identity

### When not
- internal HUDs
- repeated secondary decoration
- modest mobile without a clear budget

## 2. Portals

### What they really are
A portal is usually not a reflection. It is a window into another coherent view of the world.

The `webgl_portal` example leaves a fairly refined pattern:
- one target per visible portal
- a specific portal camera
- transform the player/camera position into the other portal’s space
- adjust the projection to fit the portal frame exactly
- hide the portal itself during its render

### What actually complicates it
- spatial correspondence between portal A and portal B
- projection fitted to the frame, not just any camera
- local clipping
- potential recursion if one portal sees another
- render order that is very easy to break

### Healthy defaults
- start with non-recursive portals
- limit recursion depth if it appears
- use moderate resolution
- hide the portal surface during its own pass
- measure spikes very early if two or more portals are on screen

For recursion, resolution by level, and finer masking, see `portal-recursion.md` and `portal-masking-stencil-scissor.md`.

### Typical risks
- looking good in a simple demo and breaking when real gameplay is added
- jitter or seams from bad camera transforms
- explosive cost if each portal renders too much world
- accidental visual recursion

### Useful levers
- target resolution
- maximum number of active portals
- limit visible content with layers or proxies
- stop updating if the portal is offscreen or irrelevant

## 3. Minimap

### What it really is
A minimap usually does not ask for cinematic fidelity. It asks for:
- clear readability
- stable orientation
- low cost

It normally fits better with an orthographic or nearly orthographic camera than with dramatic perspective.

### Recommended default
- orthographic camera for a tactical map or clean top-down view
- modest-resolution target
- update at a lower frequency than the main view if the game tolerates it
- filtered content: only what helps readability

### What to render
Do not put the whole world in without discrimination.

Better include:
- base terrain or simple proxies
- player
- objectives
- relevant enemies
- markers and navigable elements

Better exclude:
- fine cosmetic detail
- particles
- expensive transparencies
- irrelevant props
- postprocessing

### Good visual decisions
- clear colors by faction or category
- simple icons or proxies
- controlled rotation: either rotate the map or rotate the player icon, not both without a need

### Update frequency
A minimap often does not need 60 fps.

Healthy options:
- every N frames
- only when the player or targets change enough
- full update at important moments and partial update the rest of the time

If there is also fog of war, explored state, or mask blending, see `minimap-fog-of-war.md` and `fog-mask-blending.md`.

## Quick comparison between families
### Mirror
- priority: visual credibility
- camera: reflected
- risk: recursion and resolution cost

### Refractor
- priority: credible distortion/refraction
- camera: copied virtual camera + clip plane
- risk: GPU cost + expensive shader + readability loss

### Portal
- priority: spatial coherence
- camera: transformed between spaces
- risk: recursion, clipping, render order

### Minimap
- priority: readability and low cost
- camera: orthographic is almost always defensible
- risk: over-rendering useless detail

## Quality tiers by family
### Mirrors
- lower resolution first
- then lower frequency or disable secondary mirrors

### Refractors
- lower resolution first
- then disable secondary surfaces
- avoid premium shaders in modest tiers

### Portals
- limit active portals
- lower resolution
- trim visible content
- cap recursion by tier

### Minimap
- lower target size
- lower frequency
- simplify rendered layers
- separate base map and fog overlay if it exists

## Lifecycle by family
All need a clear owner, but:
- mirrors and portals usually require more care with auxiliary camera and render state
- minimap usually requires more care with content filters and associated UI

Always:
- explicit resize if it depends on the viewport
- `dispose()` the target
- cleanup auxiliary materials/surfaces if they were created for that family

## What to measure in benches
### Mirrors
- cost per active mirror
- impact of target resolution
- resize spikes

### Refractors
- cost per active surface
- impact of target resolution
- extra cost of the distortion shader

### Portals
- cost per visible portal
- impact of limited recursion
- spikes when crossing or activating portals

### Minimap
- cost by update frequency
- impact of full vs simplified layers
- visual clarity versus cost

## Anti-patterns
- using the same camera recipe for all three families
- full-res by default
- not hiding mirror/portal during its own pass
- rendering the whole world in a minimap
- assuming portal = mirror with a different texture

## Strong recommendation
Treat each family as its own subsystem:
- `mirrorSystem`
- `portalSystem`
- `minimapSystem`

Each with its own camera/math, resolution policy, update frequency, content filters, and explicit lifecycle.

## To expand
- mirrors/refractions on mobile
