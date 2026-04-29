# World Generation

## Objective
Build large or procedurally rich worlds in Three.js without falling into the classic mistake of representing every piece as a loose mesh or trying to keep everything alive at once.

## Main rule
Separate **world data** from **renderable representation**.

The world is not the scene. The scene is only a temporary, optimized view of part of the world.

## Base principle
Think in layers:
1. world **data**
2. **generation** or loading
3. **meshing / representation**
4. **streaming and disposal**
5. **gameplay over the world**

## Strong lesson from the manual
If there are many elements:
- do not create one mesh per piece by default
- do not abuse the scene graph as a data structure
- use combined geometry, instancing, or case-specific meshing

## Chunking
For large worlds, use chunks or cells.

Reasons:
- limit active memory
- rebuild only affected zones
- make streaming and disposal easier
- avoid thinking about the whole world at once

Practical rule:
- define a reasonable chunk size
- keep the logical chunk identifier separate from its visual representation
- be able to regenerate one chunk mesh without rewriting the whole world

## Voxel worlds
The voxel geometry manual leaves a very clear rule:
- simply merging cubes naively is not enough
- generate only visible faces

Key patterns:
- store voxel world data by cells
- query neighbors to decide whether a face exists
- generate custom geometry instead of instancing one cube per voxel
- avoid reserving huge memory for empty space when possible

## Heightmaps and surfaces
For heightmap-style terrain:
- a surface mesh can be enough
- raycasting against the terrain is useful for placement, navigation, or debug
- visual impact helpers help a lot with understanding what is happening

## Horizon relief as silhouette
Frequent problem: the playable area is flat (or almost), but the horizon looks empty and the sky dies against the ground. Adding a fully authored heightfield is overkill; what you want is **decorative silhouette** around the playable area, not navigable terrain.

Common failed patterns:
- **Cardinal patches**: four rectangular bumps at N/S/E/W. It is obvious they are four separate objects, the rest of the horizon stays flat, and when the player turns the camera the illusion breaks.
- **Simple radial patches**: a `height(r) = bump(r)` function produces a perfectly symmetric donut-like ring, also readable as artificial.

Reusable pattern for continuous silhouette:
1. **Radial mask**: over a large plane concentric with the playable area, `heightMask(r)` is 0 inside the gameplay radius, rises to a crest at `r ≈ rPlay + offset`, and falls again before the plane edge so it does not lift the visible skin of the world.
2. **Angular modulation** (low-freq): a `peaks(θ)` term with 2-4 peaks per revolution breaks the symmetry of the ring and creates valleys between peaks, which makes the eye read it as a mountain range instead of a donut.
3. **Noise detail** (high-freq): `fbm(x, z)` or similar adds irregular silhouette without dominating the shape.
4. **Final height**: `h(x,z) = radialMask(r) · peaks(θ) · detail(x,z)`, clamped to ≥ 0.

Rules:
- Separating the three ingredients (radial mask, angular pattern, detail) makes the result easy to tune; mixing everything into one noise function ends in magic numbers.
- The crest should fall in a band where **no gameplay system steps on the terrain** (flat collider inside `rPlay`, shadows and navigation assume Y=0). It is render-only.
- It works the same with a subdivided `PlaneGeometry` writing Z into vertices, or with an authored heightfield; choose based on how much control you want.
- If **the terrain is part of gameplay** (collisions, player height, navmesh), this pattern does not apply: use an authored heightfield / real bake. This is a pattern for flat worlds that only need background.

When it is enough:
- top-down / third-person games with a closed playable area
- the horizon is far away and small on screen
- there is no free camera that lets the player see it from above

When to escalate to authored heightfield:
- the camera can see the relief from angles that expose its geometry
- mechanics depend on height (climbing, rolling, water)
- the author wants specific art direction ("a ridge here, a canyon there")

## Combined geometry vs instancing
### Combine geometry
Good option when:
- there are many static elements
- individual pieces do not need to change often
- you want to reduce draw calls as much as possible

### Instancing
Good option when:
- there are many similar elements
- you do want some per-instance change
- you need to update transforms or colors without rebuilding the whole mesh

### Loose meshes
Keep them for:
- truly special objects
- important interactive entities
- small quantities

## Positioning helpers
The manual teaches a fine pattern: use a few temporary `Object3D` helpers to calculate complex positions instead of flooding the scene graph with persistent nodes.

Rule:
- use helpers for calculation
- do not turn them into a permanent structure if not needed

## Streaming
A large world should assume that:
- chunks enter
- chunks leave
- resources are released
- the visible scene changes

Useful questions:
- which chunks should be active around the player?
- what active distance do we use for visuals, physics, and gameplay?
- when do we regenerate mesh and when do we only update data?

## Anti-patterns
- one mesh per block or tiny prop in huge worlds
- using the scene as the world database
- not separating logical world and visual representation
- not chunking when the world is already screaming for it
- rebuilding the whole world for small local changes

## Strong recommendation
For web games:
- chunked logical world
- aggregated representation per chunk
- instancing for repetition with light variation
- individual meshes only for important objects

## To expand later
- chunk size policy
- world streaming around camera or player
- incremental meshing
- integration with physics by chunk
- navigation and spatial queries
- reproducible procedural generation by seed
