# Transparency Pitfalls

## Goal
Avoid one of Three.js’s classic traps: assuming transparent materials will behave intuitively just because you set `transparent = true`.

## Main rule
**Realtime transparency is not magic.**
Render order, depth, and sorting matter a lot, and there are cases where no cheap perfect solution exists.

## What usually goes wrong
- transparent objects draw in a weird order
- back or internal faces disappear or flicker
- glass or overlays cover things incorrectly
- stacked particles and transparencies look bad
- postprocessing worsens artifacts

## Healthy mental model
With opaque objects, the depth buffer helps a lot.
With transparent objects, Three.js usually relies heavily on object sorting, and that has clear limits.

Result:
- between complex transparent objects, order can fail
- inside the same mesh, the problem can be even worse

## Healthy defaults
Before reaching for weird hacks:
- ask whether real transparency is truly needed
- prefer opaque, alpha test, or dither if that is visually enough
- keep few simultaneous transparent layers
- avoid complex interpenetrating transparent geometry as the game’s foundation

## Healthier alternatives
### 1. Alpha test / cutout
Useful for:
- leaves
- fences
- cutout sprites
- details where smooth semitransparency is not needed

Advantage:
- much more stable than classic transparency

### 2. Fake transparency
Useful for:
- diegetic HUDs
- stylized effects
- surfaces where the feeling matters more than physical correctness

### 3. Dither / temporal tricks
Sometimes fits better than stacking expensive, fragile transparent materials.

## If real transparency is needed
Look at these levers:
- `depthWrite = false` often helps
- `depthTest` depending on the case, carefully
- `renderOrder` for concrete, controlled cases
- split geometry into separate layers or meshes
- simplify the shape or number of visible layers

## What not to do
- use `renderOrder` as a universal hammer
- assume one transparent material fixes complex glass, particles, and overlays at the same time
- mix too many transparent surfaces while expecting perfect correctness
- add transparency for fun when alpha test or opaque materials solved the problem

## Typical cases
### Foliage
- almost always better with alpha test than smooth transparency

### Glass
- use in moderation
- split pieces if needed
- confirm whether the visual effect is worth the cost and artifacts

### Particles
- control blending and number of layers
- assume heavy overlap will bring visual and performance problems

### UI in a 3D world
- try simple composition and layers
- do not treat it as physically correct glass

## When to prototype first
Do an early spike if the game depends heavily on:
- lots of glass
- dense particles
- hero semitransparent materials
- complex composition with postprocessing

## Strong recommendation
First decide whether you need:
- opaque
- alpha test
- fake transparency
- real transparency

Then choose the cheapest option that preserves visual readability.

## Related references
- `render-targets.md`
- `postprocessing.md`
- `quality-tiers.md`
