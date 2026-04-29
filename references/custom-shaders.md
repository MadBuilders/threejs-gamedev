# Custom Shaders

## Goal
Write and maintain custom shaders in Three.js without throwing away the engine’s material/lighting system, and knowing when a standard material is enough.

## Main rule
**Do not write a shader until you have ruled out `MeshStandardMaterial` with well-made textures and a little controlled vertex displacement.**
Many effects that seem to require a shader are solved with textures, masks, and simple uniforms.

## When a shader is worth it
- time-dependent effects (dissolve, hologram, shimmer)
- non-physical shading (toon, cel-shade, paper, comic)
- dynamic geometric distortion (waves, wind, jelly)
- blending by world rules (triplanar, slope-aware terrain, height)
- impostors, advanced billboarding, custom particle FX
- custom postprocessing not covered by any standard pass

## When it is not worth it
- changing base color → `material.color`
- making something “shiny” → adjust `metalness`/`roughness` and lighting
- a simple outline → postpro or double pass, not an object custom shader
- a vertical gradient → `vertexColors` or a texture
- distance fade → engine fog or material property

## Material choice
Three broad paths:

### 1. `onBeforeCompile` on a standard material
- you keep the engine’s lighting, shadows, tonemapping, and everything else.
- you inject uniforms and modify specific chunks of the generated shader.
- ideal for vertex displacement on `MeshStandardMaterial` without losing PBR.
- risk: coupling to internal chunks that may change between Three.js versions.

### 2. `ShaderMaterial` / `RawShaderMaterial`
- total control.
- you lose the engine’s lighting chain unless you reimplement it.
- good for unlit effects, postpro, fullscreen passes, very specific things.
- `RawShaderMaterial` does not add any uniform/attribute automatically: you do the work yourself.

### 3. Node-based (`NodeMaterial`, TSL)
- modern, modular API, portable between WebGL2/WebGPU.
- useful for projects targeting WebGPU or wanting shader editing by composition.
- younger, fewer examples in the wild, may change.
- worth considering for new projects intended to last for years.

## Common patterns

### Healthy vertex displacement
- use `onBeforeCompile` on `MeshStandardMaterial`.
- inject `uTime` uniform and noise/curl functions.
- keep `normal` consistent: if you displace the vertex, recalculate or approximate the normal if you want lighting not to lie.
- avoid expensive 3D noise if 2D is enough.

### Fullscreen passes
- fullscreen quad with `ShaderMaterial` and a trivial orthographic camera.
- use RTT with controlled resolution and frequency (see `render-targets.md`).
- split passes if it helps readability, even if you add an intermediate target.

### Terrain blending (slope/height/triplanar)
- sample textures by world component, not exclusively by UV.
- prebaked or procedural masks, not hardcoded.
- compact atlases if there are many material variants.

### Dissolve / reveal
- noise texture + animated threshold.
- `discard` for cutout, but be careful: `discard` disables optimizations (early-z) and can cost more than it seems, especially on mobile.

### Water / waves
- displacement with summed `sin/cos` or vertex noise.
- reflection/refraction with RTT (see `render-target-families.md`).
- animated normal map in fragment for detail without inflating vertices.

## Uniforms and state
- shared `uniforms` objects when several materials use the same time/params.
- update in a central system (`uniformsUpdater`), not in each entity.
- avoid creating new objects every frame (`new THREE.Vector3(...)` in update is a constant garbage drip).

## Precision
- `mediump` on mobile by default, `highp` where needed (depth, normals in serious shading).
- do not assume `highp` always exists in mobile fragment shaders.

## Defines and variants
- `#define` by capability (`USE_NORMALMAP`, `ANIMATE_VERTICES`) to compile only what is needed.
- beware combinatorial variant explosion: if there are too many, move to boolean uniforms even if you pay some runtime cost.

## Shadows and shading with custom vertex
If you displace vertices in `onBeforeCompile`:
- projected shadows are calculated with a separate material for the shadow pass.
- apply the same displacement to the mesh’s `customDepthMaterial` and `customDistanceMaterial` so the shadow does not lie.
- alternative: avoid shadows on meshes with strong displacement.

## Custom postpro
- small composed passes instead of a mega-shader.
- measure with `benchmarking.md`: a custom pass is often cheaper than expected, or the opposite, much more expensive.
- on mobile, every extra pass is noticeable.

## Shader debug
- “debug mode” uniform that paints normals, UVs, depth, mask.
- isolation view: flat material with only the part you are unsure about.
- `console.log(material.program?.fragmentShader)` carefully; this is for reading during development.
- integrate browser extensions (Spector.js) when captures are needed.

## Cross-version
- Three.js internal chunks change. If you use `onBeforeCompile`, pin the Three.js version and review when updating.
- keep minimal visual tests (screenshot or verification scene) to catch breakage quickly.

## WebGPU / TSL
If the project may target WebGPU later:
- prefer `NodeMaterial` from the start when it makes sense.
- isolate shader logic in modules to ease migration.
- do not invest heavily in manual shaders tightly bound to WebGL 2.

## Anti-patterns
- writing `ShaderMaterial` for something a textured `MeshStandardMaterial` solves
- unnecessary `discard` in fragment, blocking optimizations
- recalculating static uniforms every frame
- mega-shader with every branch, mixing effects that are not always used
- injecting `onBeforeCompile` without pinning the Three.js version
- using `highp` indiscriminately on mobile
- displacing vertices without correcting the shadow pass
- forgetting `needsUpdate` when changing defines

## Strong recommendation
Healthy flow:
1. can I do it with a standard material + texture?
2. if not, is `onBeforeCompile` enough?
3. if not, isolated `ShaderMaterial`, with centralized uniforms and documentation of the chunk/version.
4. measure cost with a small bench before adopting it as a default.

## Related references
- `lights-shadows.md`
- `transparency-pitfalls.md`
- `postprocessing.md`
- `render-targets.md`
- `benchmarking.md`
- `mobile-performance.md`
