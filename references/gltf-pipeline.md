# glTF Pipeline

## Goal
Treat `glTF` and `GLB` as a real production pipeline for Three.js, not as a simple isolated `loader.load()`.

## Primary default
For new web games in Three.js:
- use `glTF` or `GLB` as the main runtime format
- keep source files separate
- centralize loading with `GLTFLoader`
- use `LoadingManager` when there are multiple assets or a real loading screen
- consider geometry and texture compression only with a clear pipeline

## Main rule
Separate these layers:
1. **source assets**
2. **runtime exports**
3. **load orchestration**
4. **instancing/cloning**
5. **visual activation and cleanup**

## `glTF` vs `GLB`
Practical rule:
- `GLB` is usually the better distribution default when you want a packaged runtime that is simple to serve
- `glTF` can be useful when you need easy inspection or explicit external assets

Do not turn this into religion. What matters is that the runtime is consistent and maintainable.

## Source files vs runtime files
Strong rule:
- editor files are not runtime assets
- export a version intended for the game
- do not depend on the Blender, Maya, or similar file as if it were the final asset

Keep these clear:
- editable source
- runtime export
- compressed variant if one exists

## Export checklist
Before integration:
- correct scale
- correct orientation
- useful pivots
- stable names
- reasonable materials
- clean hierarchy
- named animation clips
- sensibly sized textures
- polycount proportional to actual use

## Animation: beware **scale** tracks

In rigs exported from AI tools or noisy retargeting, a clip may include **scale keyframes** on root or torso bones. During playback this becomes “inflation” or clipping during the walk.

Options:
- fix it in DCC / re-export cleanly;
- at runtime, **remove scale tracks** from the `AnimationClip` on load (leaving position and rotation) if the model already has correct scale in bind pose.

Related: `animation-systems.md` and bounding boxes on skinned meshes (`Box3.setFromObject(..., true)`).

## Load orchestration
When there are multiple models or dependencies:
- use `LoadingManager`
- expose progress to the user if the wait is not trivial
- do not start serious gameplay before critical assets are ready

The `game` manual is quite clear here: load coordination and progress UI are part of the product, not a minor detail.

## Recommended asset registry
Healthy pattern:
- register assets by id
- separate metadata from runtime object
- cache the load result when appropriate
- expose factories for visual instances

Conceptual example:
- `assets.characters.knight`
- `assets.props.crate`
- `assets.environments.village`

## Cloning and instancing
Not every loaded asset should be added to the scene as-is.

### Case 1, one large scene
- load
- mount
- configure ownership and lifecycle

### Case 2, multiple instances of the same asset
- clone deliberately
- if it is a skinned character, use `SkeletonUtils.clone()`
- do not accidentally share animated state

### Case 3, many repeated objects
- evaluate instancing or assets prepared with instancing
- do not assume cloning hundreds of normal nodes is free

The `webgl_loader_gltf_instancing` example leaves a useful hint: glTF can coexist with `EXT_mesh_gpu_instancing`, so part of the cost can already be solved in the asset pipeline.

#### Migrating from `clone()` to `InstancedMesh` per leaf
Recurring pattern: a `loadXxxModel()` function returns a wrapper exposing `instance()` implemented as `wrapper.clone(true)`. That works until the level scales to dozens or hundreds of copies of the same GLB: each instance adds its own `Object3D`s (transform, matrix world update) and one draw call per leaf mesh of the wrapper.

Reusable template to extend the same factory with real instancing, without forcing the rest of the game to change:

```ts
// 1) When loading the GLB, extract the flattened leaf meshes with their matrix
//    local to the wrapper (the one that already applies scale, recentering, etc.).
const leafMeshes: { geometry: THREE.BufferGeometry; material: THREE.Material; baseMatrix: THREE.Matrix4 }[] = [];
wrapper.updateMatrixWorld(true);
wrapper.traverse((obj) => {
  const mesh = obj as THREE.Mesh;
  if (!mesh.isMesh) return;
  leafMeshes.push({
    geometry: mesh.geometry,
    material: mesh.material as THREE.Material,
    baseMatrix: mesh.matrixWorld.clone(),
  });
});

// 2) New method: one InstancedMesh per leaf, N matrices.
function createInstancedMeshes(placements: ReadonlyArray<{ x: number; z: number; yaw: number }>) {
  const group = new THREE.Group();
  const tmp = new THREE.Matrix4();
  const placement = new THREE.Matrix4();

  for (const leaf of leafMeshes) {
    const im = new THREE.InstancedMesh(leaf.geometry, leaf.material, placements.length);
    im.castShadow = castShadow;
    im.receiveShadow = true;
    // With instances scattered across the whole world, per-instance frustum culling
    // can hide real instances; it is usually worth leaving this to the global
    // bounding sphere.
    im.frustumCulled = false;

    for (let i = 0; i < placements.length; i++) {
      const p = placements[i]!;
      placement.makeRotationY(p.yaw).setPosition(p.x, 0, p.z);
      tmp.multiplyMatrices(placement, leaf.baseMatrix);
      im.setMatrixAt(i, tmp);
    }
    im.instanceMatrix.needsUpdate = true;
    group.add(im);
  }
  return group;
}
```

Things to watch:
- **Group placements by variant** before calling the instanced factory: if your game mixes “tree A” and “tree B”, those are two different `InstancedMesh` objects (do not force them into one even if they share scale).
- **Keep `instance()` for rare cases**: interactive obstacles, props that need their own animation, or material swaps; those still use `clone(true)`. The instanced path is the default for “dense decoration”.
- **Collision obstacles that pointed at the individual instance should now point at the whole instanced group** (if you need them). Check where that `visual` is used (in many games only for debug or for a one-off swap); if the only lifecycle is “lives from boot until the end of the level”, sharing the reference is safe.
- **Draw calls**: you go from `N × leafCount` to `leafCount` (one per GLB submesh), not “to 1” unless the GLB has a single mesh.
- **Shadows**: `InstancedMesh` casts shadows per instance like a normal mesh; the shadow pass also benefits from the draw call collapse.
- If the GLBs already carry `EXT_mesh_gpu_instancing` from the source, this runtime step is unnecessary: the pipeline already solves it.

## AI-generated GLBs (Meshy and similar)

AI 3D asset generators (Meshy, Rodin, Luma, etc.) speed up lookdev a lot, but **they come with silent defaults that must always be corrected** before putting the GLB in the game. Treat them as “editable source with nice lookdev”, not as runtime assets.

Mandatory checklist after downloading any AI GLB:

- **`doubleSided: true` on all materials**, even opaque ones (trunks, walls, vases). For closed geometry this is pure fragment shader cost, roughly 2×, with no visual gain. Force `FrontSide` — see below.
- **2048×2048 textures by default**, every PBR slot (baseColor + normal + metallicRoughness + emissive). A single tree can ask for ~90 MB of VRAM. Drop to 512 or 1024 depending on actual use.
- **No geometry or texture compression**. Add `meshopt` + `webp`/KTX2 as an automatic pipeline step.
- **Uncontrolled polycount**. A “presentation” mesh can arrive at 0.5M–3M tris. For realtime: distant foliage 10K, scenery prop ~10–30K, building ~20–30K. Go higher only if the silhouette truly demands it.
- **Skinned meshes: do not run through AI remesh**. AI tools break weights, skeleton, and animations. If you need to reduce an animated character, the path is Blender Decimate while preserving vertex groups, then reverify every clip.

### Canonical post-download pipeline

```bash
# Downscale textures to their real in-game usage size
pnpm dlx @gltf-transform/cli@latest resize in.glb tmp.glb --width 512 --height 512

# Re-encode to WebP (normal maps and the rest). KTX2 if the runtime supports it.
pnpm dlx @gltf-transform/cli@latest webp tmp.glb tmp2.glb

# Geometry compression
pnpm dlx @gltf-transform/cli@latest meshopt tmp2.glb out.glb
```

Then run `inspect` to confirm the result is what you expect (tris, `doubleSided`, texture resolution, applied extensions).

### Force `FrontSide` in the loader, not in the GLB

Fixing `doubleSided` with `gltf-transform` or re-export also works, but **it is more robust to do it in the game loader**:

```ts
gltf.scene.traverse((obj) => {
  const m = obj as THREE.Mesh;
  if (!m.isMesh) return;
  const mats = Array.isArray(m.material) ? m.material : [m.material];
  for (const mat of mats) {
    if (mat) mat.side = THREE.FrontSide;
  }
});
```

Advantage: it covers any future re-export (another generation, another tool, another artist) without depending on someone remembering to run the pipeline. One control point in the loader per asset type (trees, props, houses) prevents the “optimization” from being tied to one concrete GLB.

### Variant scale as authored data, not baked into the GLB

If an asset will be regenerated (new texture, better mesh, stylistic variant), **do not bake the final size into the GLB** — keep it as a constant in code/JSON:

```ts
export const TREE_VARIANT_SCATTER_SCALE: Record<TreeVariantKind, number> = {
  olive: 1.0,
  poplar: 1.3,
  'poplar-alt': 1.8,
};
```

This lets you re-remesh the asset without retuning placements across the whole scene. It is the same **“editable source → artifact”** doctrine applied to dimensions: the GLB is the artifact; decisions about “how big it is in the world” are human source.

Applicable to trees, props, buildings, and anything whose mesh you may want to refresh in the future.

## Compression
The official review pushes several different ideas:

### HTTP compression
First almost-free win:
- serve assets with correct HTTP compression
- often gives a huge improvement without touching the asset content

The `game` manual gives a very clear example: several megabytes shrink massively just through server compression.

### Geometry and texture compression
The `webgl_loader_gltf_compressed` example shows a canonical path:
- `KTX2Loader` for compressed textures
- `MeshoptDecoder` for compressed geometry

Base pattern:
- detect the renderer’s real KTX2 support
- configure auxiliary loaders explicitly
- do not add compression blindly without validating the pipeline and target devices

## gltf-transform (CLI)

[gltf-transform](https://gltf-transform.dev/) is the reference tool for **inspecting, cleaning, and optimizing** glTF/GLB in a reproducible pipeline. It does not replace Blender/Substance for authoring, but it does replace **ad hoc export** and “shrinking megabytes” before uploading to `public/`.

**Package:** `@gltf-transform/cli` (the binary is usually invoked as `gltf-transform` via `npx` or `pnpm dlx`).

**When to use it**
- Before integrating a huge GLB: understand what is heavy (geometry vs textures vs extensions).
- Before production: deduplicate accessors, simplify materials, compress geometry (for example Meshopt) or textures according to the project.
- In CI: validate that an export has not grown beyond a threshold (combine with `benchmarking.md` if applicable).

**Typical commands (examples)**

```bash
# Structure, sizes, meshes, animations, textures
npx @gltf-transform/cli inspect model.glb

# General optimization (review flags in the package docs; they evolve between versions)
npx @gltf-transform/cli optimize input.glb output.glb
```

**Healthy rules**
- Pin the CLI’s **major version** in the project (script in `package.json` or documented in README) so `optimize` is reproducible across machines.
- After optimizing, **test in the real game** (Three.js + extensions you use: Meshopt decoder, KTX2, etc.).
- Do not treat compression as magic: if the asset is still huge, the bottleneck is often **4K textures** or DCC export options.

**Anti-patterns**
- Optimizing once “by hand” without a script or pinned version, then forgetting how the artifact was regenerated.
- Assuming `optimize` always lowers visual quality: it depends on flags and content.

## Current recommendation
Based on this review wave, the healthiest bet for serious projects would be:
- `GLB` as the main runtime artifact
- HTTP compression whenever possible
- `KTX2` for textures if the pipeline supports it well
- `Meshopt` as a very serious option for geometry
- Draco only if it fits your pipeline and you have measured it, not by reflex

## Shader warmup and visual activation
The modern `webgl_loader_gltf` example leaves a very valuable detail:
- `renderer.compileAsync()` before adding the model can prevent visible stalls when activating the asset

This deserves to be the mental default in scenes where you:
- load on demand
- change character or skin
- enter new areas
- present large models to the user

For frame pacing, warmup, and general hitch-free activation policy, see `frame-pacing-stutter.md`.

## Environment and lookdev
Several official examples mix glTF loading with:
- environment map
- tone mapping
- camera adjustment

This matters because an asset may “look bad” not because of the asset, but because of:
- an environment that does not light it well
- incoherent tone mapping
- a poorly adjusted camera

Do not blame the model too early.

## Camera fit and presentation
For viewers, selection menus, or inspection:
- compute bounds
- fit the camera to the selection
- update near/far deliberately

The official example uses this well as a presentation pattern.

## Lifecycle of loaded models
When changing models or replacing views:
- remove the visual from the scene
- stop actions or mixers if they exist
- clean up resources if the asset will not be reused
- do not let old loads win async races

The modern official example also leaves a useful signal:
- use ids or load guards to ignore old responses when the user changes assets quickly

## Ownership
Every loaded model should have a clear owner:
- viewer
- scene chunk
- enemy factory
- character roster
- skin selector

Without clear ownership, chaos enters through three places:
- leaks
- duplicate loads
- broken cleanup

## Anti-patterns
- treating the DCC export as the final asset without review
- loading glTF from any file without a registry or coordination
- mixing load, gameplay, and visual setup in the same huge callback
- cloning animated characters without `SkeletonUtils.clone()`
- compressing assets without a reproducible pipeline
- ignoring shader compilation stutter
- having no strategy for logical cancellation or obsolete loads

## Strong recommendation
If the project grows beyond a small prototype, explicitly create:
- `assetRegistry`
- `gltfAssetLoader`
- `modelFactory`
- `preloadPhase` or `loadingScreenController`
- `assetLifecycle` or integration with the general lifecycle

## To expand
- variants by platform
- preload by scene or biome
- glTF bundle streaming
- automatic budget validation
- exact Draco vs Meshopt policy per project
