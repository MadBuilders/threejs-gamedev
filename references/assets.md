# Assets

## Goal
Define a practical 3D asset pipeline for pure Three.js games, focused on stability, clarity, and reasonable cost for the web.

## Primary default
Use `glTF` or `GLB` as the main format for 3D assets.

Reasons:
- it is the most natural format for modern Three.js
- it carries meshes, materials, hierarchy, animations, and scenes
- it reduces weird conversions halfway through the project
- it fits well with web workflows and current tools

## Base rule
Split the pipeline into four phases:

1. **generation or acquisition**
   - manual modeling
   - external libraries
   - generative tools like Meshy when they fit
2. **cleanup and validation**
   - scale
   - orientation
   - names
   - polycount
   - materials
3. **compression and packaging**
   - decide whether Draco or KTX2 adds real value
4. **game integration**
   - loading
   - cache
   - instancing
   - binding to gameplay

## Recommended formats

### 3D
- default: `glTF` / `GLB`
- avoid legacy formats unless there is a specific need

### Textures
- prefer reasonable sizes
- avoid giant textures by default
- use compression when the pipeline allows it
- keep clear color and map conventions
- for map details, color space, tiling, anisotropy, and compression, see `texturing-pipeline.md`

### Audio
- outside the main scope of this reference for now

## Base loaders
Usually start with:
- `LoadingManager`
- `GLTFLoader`
- `TextureLoader`
- additional loaders only when justified

For the specifics of export, `GLB` vs `glTF`, compression, `compileAsync`, async guards, and instancing variants, see `gltf-pipeline.md`.

The manual review reinforces a clear decision: for new games, treat `glTF` as the happy path and avoid opening too many fronts with legacy formats unless there is a real need.

## Integration rules
- Do not load assets from anywhere without coordination.
- Centralize paths, preload, and load errors.
- Separate asset loading from gameplay logic.
- Do not assume an external asset arrives clean.
- Create wrappers or factories when an asset has repeated configuration.
- Have a resource release strategy for when an asset is no longer used.

## Loading and lifecycle
The manual pushes an important idea well: loading is only half the problem. The other half is knowing when to keep, reuse, or release resources.

Practical rule:
- if an asset is reused heavily, cache it deliberately
- if it belongs to an area or scene that disappears, prepare to unload it
- do not leave geometries, materials, and textures alive by accident for the whole session

The instancing/performance example also gives another healthy signal: when you rebuild large mesh groups or change representation strategy, explicitly clean up old geometries and materials instead of trusting the problem to fix itself.

And the disposal manual makes it even clearer: removing a mesh from the scene does not automatically release geometry, material, or texture. If an asset is no longer needed, there must be a real cleanup path.

## “Placeholder first, swap later” pattern
A pattern that repeats with all heavy assets (GLBs, HDRIs, PBR textures): **do not block game startup on a download**. The game starts with an acceptable placeholder, and the real asset is hot-swapped in when it resolves. This applies equally to:
- models → simple primitive (capsule, box) + `setVisual(real)` when the GLB loads
- materials → `MeshStandardMaterial` with a flat color, then `mesh.material = real`
- environments/skybox → flat background color, then `scene.background = envTex`

Rules:
- the placeholder must be playable, not “broken with a black screen”
- the swap must be precise and have a clear handle (`setVisual`, `swapMeshMaterial`, a `loadX(level)` function that performs the swap)
- when replacing, **release the old resource**: `oldMaterial.dispose()`, `oldTexture.dispose()`, `oldGeometry.dispose()`. Removing an object from the scene releases nothing by itself.
- if loading fails, `console.error` and keep playing with the placeholder; never throw

This pattern also cleans up the development cycle: code changes are visible instantly without waiting for every current GLB/HDRI to reload.

## Parallel prefetch for prop collections
When a level places many copies of a few “kinds” (houses, trees, rocks, props with one factory per kind), the natural loop is:

```ts
for (const placement of level.props) {
  const model = await getModel(placement.kind);   // ← serializes fetches
  addInstance(model, placement);
}
```

If the loader already caches by kind, this is still serial the first time each kind appears: the first prop of each type blocks the next one until its GLB resolves. With 4–5 different kinds and mediocre network, you can lose several seconds of boot time unnecessarily.

Generic trick: **preload the unique kinds in parallel** before the placement loop.

```ts
const uniqueKinds = Array.from(new Set(level.props.map((p) => p.kind)));
await Promise.all(uniqueKinds.map((k) => getModel(k)));

for (const placement of level.props) {
  const model = await getModel(placement.kind);   // ← now an instant cache hit
  addInstance(model, placement);
}
```

Requirements for this to stay clean:
- the loader must **cache by kind**, or you will fetch twice.
- the `await` inside the loop is still useful: if the cache failed for any reason, it keeps loading instead of crashing.
- if there are very many unique kinds (hundreds), `Promise.all` is not the pattern; you want a pool with limited concurrency (for example, 4–8 simultaneous fetches).

This same pattern applies to HDRIs, sprites, and any resource with a cached factory.

## Drama-free disposal
When you replace a resource, the old one is not released by itself. Mini-recipes:

- **material**: `oldMat.dispose()`. If that material had exclusive textures, dispose those too (`oldMat.map?.dispose()`, etc.).
- **texture**: `tex.dispose()`.
- **geometry**: `geom.dispose()`.
- **render targets**: `rt.dispose()`. `PMREMGenerator` exposes its own `dispose()` after using `fromEquirectangular`.
- **scene objects**: `scene.remove(obj)` + traverse and dispose each descendant mesh’s `geometry`/`material`.

Broad rule of thumb: if you duplicate assets and do not see GPU memory drop after reloading, suspect a forgotten `dispose()`.

## Intake checklist for a 3D asset
Before putting an asset into the game, check:

- **scale**: it is not absurdly large or small
- **orientation**: forward/up match the game
- **pivot**: it makes sense for animation and placement
- **polycount**: it is not disproportionate to its actual use
- **materials**: it does not come with impossible or overly expensive materials
- **textures**: sizes, compression, names, and maps are correct
- **hierarchy**: clean nodes, no unnecessary junk
- **animations**: clear names and useful clips
- **shadows**: check whether it should actually cast/receive shadow

## Recommended conventions
- stable names for nodes and clips
- folders by asset type
- clearly distinguish source files from runtime files
- keep a list of heavy or problematic assets
- version pipeline decisions if they change during the project

## Meshy and generative tools
They can be used to speed things up, but with discipline.

Treat Meshy as:
- a prototyping accelerator
- an option for concepts or secondary props
- a tool that requires later manual review

Do not treat Meshy as a guarantee of:
- good topology
- production-ready materials
- correct scaling
- reasonable cost for mobile

## Anti-patterns
- mixing FBX, OBJ, and other formats without a clear reason
- using generated assets without technical review
- loading each asset ad hoc in scattered files
- giant textures because “they look better”
- solving pipeline problems late, after 40 assets are already in the project
- having no plan for `dispose()` and resource cleanup
- blocking boot while waiting for a “pretty” asset: it breaks the dev loop and leaves the player on a blank screen if it fails
- replacing materials/textures without disposing the old ones (GPU memory silently leaks)

## To expand
- preload and asset registry
- asset streaming by areas or chunks
- automatic validation of names and sizes
- integration with animations and state machines
