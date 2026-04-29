# Texturing Pipeline

## Goal
Decide **which maps you use, how you configure them, and how you compress them** for pure Three.js, without turning it into theory or a 4K obsession.

## Main rule
**Fewer textures, configured better, before more textures.**
80% of the result comes from: sensible size, correct color space, filtering, and reasonable repetition. The remaining 20% is compression.

---

## Map types and when to use them

### Base / Albedo (diffuse color)
- You almost always want it.
- **Color space: sRGB.** In Three.js: `texture.colorSpace = THREE.SRGBColorSpace`.
- Do not bake lighting into the albedo if you will use dynamic lights.

### Normal
- Adds relief without geometry.
- **Color space: linear** (NOT sRGB). Do not touch `colorSpace`, or leave it as `NoColorSpace`.
- Usual convention: tangent-space, OpenGL (+Y up). If the relief looks inverted on Y, invert it in the map or flip Y in the shader.

### Roughness / Metalness / AO
- **All linear.** These are *data maps*, not colors.
- `MeshStandardMaterial` accepts combined AO/Roughness/Metalness maps in a single RGB using glTF conventions (R=AO, G=roughness, B=metalness). This saves memory and draw calls.
- If you do not have metalness, do not force the material to be metallic: floors and fabric almost never are.

### Height / Displacement
- Only if you will truly displace geometry (`displacementMap`) or use parallax.
- Using it as a disguised normal map is usually expensive and ugly.

### Emissive
- Useful for signs, lamps, emissive HUDs.
- `emissiveMap` works well with `emissiveIntensity`.

---

## Color space: the most common mistake

In modern Three.js:
- `renderer.outputColorSpace = THREE.SRGBColorSpace` (almost always).
- **Albedo/emissive** → `SRGBColorSpace`.
- **Normal, roughness, metalness, AO, mask, displacement** → `NoColorSpace` (linear).

Classic symptom when this is forgotten: washed-out colors, lighting that looks too dark or too loud, or normals that *almost* work but look muted. Check color space before touching lights.

---

## Size, repetition, and filtering

### Size
- Reasonable default for web games: **1K–2K per map**.
- 4K only when it is seen very close and justified. 4K doubles VRAM (and mipmaps).
- For small distant props: 512 and below.

### Repetition (tiling)
- `texture.wrapS = texture.wrapT = THREE.RepeatWrapping` for floors/walls.
- `texture.repeat.set(x, y)` proportional to the plane size in meters, not to the mesh geometry.
- **Single tiling is noticeable.** Mitigate with:
  - a detail map added in a custom shader,
  - decals or extra patches,
  - splat / blending (see below).

### Filtering and mipmaps
- `texture.minFilter = THREE.LinearMipmapLinearFilter` (default in Three).
- `texture.magFilter = THREE.LinearFilter`.
- `texture.generateMipmaps = true` (default for power-of-two textures).
- **Anisotropy**: raise it to the maximum supported value when the camera sees textures at grazing angles (large floor):
  ```ts
  texture.anisotropy = renderer.capabilities.getMaxAnisotropy();
  ```
- Without anisotropy, a PBR floor looks sloppy in perspective.

---

## Compression and format

### What to choose for runtime
- **PNG/JPG**: baseline. Always works, does not use GPU compression.
- **WebP**: good CPU compression, decodes to RGBA on the GPU. Good for albedo where download size matters.
- **KTX2 + Basis Universal**: native GPU compression (less VRAM). Recommended when there are many textures or modest devices (see the compression section in `gltf-pipeline.md`; requires `KTX2Loader` and renderer support).
- **Inside glTF**: use `KHR_texture_basisu` (KTX2) when the pipeline supports it well; the `gltf-transform` CLI can apply it as part of `optimize` (see `gltf-pipeline.md`).

### Practical rule
- Prototype: PNG/JPG or WebP.
- Production with many assets: KTX2 where the savings are noticeable.
- Do not compress blindly: validate visual quality in the game, not in the viewer.

---

## Single tiling vs material blending (floors, terrain)

### Single tiling
- One tileable PBR material applied to the plane.
- Simple, fast; looks obvious on large surfaces.

### Splat / texture blend
- A mask texture (RGBA or one channel per layer) decides the weight of 2–4 materials.
- Requires `onBeforeCompile` on `MeshStandardMaterial` or a custom `ShaderMaterial` (see `custom-shaders.md`, section “Terrain blending”).
- Moderate cost: more samples per fragment.

### Triplanar
- Samples along the three world components; avoids stretching on slopes.
- **Expensive on mobile**. Use only when stretching is visible and reasonable UVs are unavailable.

### Vertex colors / vertex attributes
- Alternative without extra textures for adding variance.
- Cheap but rough; good for prototypes.

---

## Performance and limits

- Every material with different `map/normalMap/...` increases memory and, especially, **draw calls**. Consider atlases or a `MeshBasicMaterial` pool when there are many simple objects.
- Mobile: stricter VRAM budget (see `mobile-performance.md`). Use 1K as the default, not 2K.
- Avoid non-power-of-two textures if you use wrapping or mipmaps: some drivers struggle; WebGL2 supports it, but cost can rise.
- `anisotropy > 1` is cheap on desktop, more expensive on mobile; lower it if fragment cost shows up.

---

## Sources (see `default-content-sourcing.md`)

- **ambientCG** (CC0) as the default for PBR materials/surfaces.
- For concept/props: Kenney, Poly Pizza, Sketchfab (with the license read carefully).
- Do not mix five styles without intent.

---

## Texture intake checklist

- [ ] reasonable size for the use (1K–2K typical)
- [ ] correct color space (sRGB vs linear)
- [ ] clear map type (albedo / normal / roughness…)
- [ ] correct wrap and repeat
- [ ] anisotropy configured on large/oblique surfaces
- [ ] mipmaps active when applicable
- [ ] distribution format decided (PNG / WebP / KTX2)
- [ ] license recorded (see `default-content-sourcing.md`)

---

## Anti-patterns

- The whole floor in 4K “just in case”.
- Albedo with baked lighting and then dynamic lights added on top.
- Normal map in sRGB: produces fake, washed-out relief.
- Obvious single tiling, “fixed” by raising the texture to 4K.
- KTX2 compression without validation in the real game.
- Textures added across a thousand files without a central registry (see `assets.md`).
- Confusing `displacementMap` with `normalMap`: the first displaces geometry and needs tessellation; the second only changes shading.

---

## Ribbon meshes over curves (paths, rivers, walls)

Recurring pattern: you want a textured strip that follows a curve in the world (a winding dirt road, river, fire trail, race track). What is not obvious is **how to hook it into the texture pipeline** so tiling looks consistent.

### Recipe

1. Define the curve with `CatmullRomCurve3` over XZ waypoints (and a small `yLift` to avoid z-fighting with the ground).
2. Sample it with `curve.getSpacedPoints(N)` — `N` ≈ 2 × arc-length in meters gives enough resolution without overdoing it.
3. For each sample, compute tangent (finite difference with the next point) and `side = up × tangent` (normalized). Build the two left/right vertices at `±width/2` from the center point.
4. **Normalized UVs 0..1** in both directions:
   - `u = 0` on the left vertex, `u = 1` on the right.
   - `v = arc[i] / totalLength`, with `arc[i]` accumulated in the first pass.
5. Triangle-strip indices: `a,c,b` + `b,c,d` per segment.
6. Material: `MeshStandardMaterial` with PBR maps in `RepeatWrapping`.
7. Final tiling: `texture.repeat.set(tilesPerMetre * width, tilesPerMetre * length)`. This way the texture sees exactly the number of repetitions matching the real ribbon size, and if you change the geometry (add a waypoint), tiling recalculates by itself with the updated `length`.

### Why normalized UVs and not “in meters”

Temptation: set `uv.v = arc_in_metres` directly and leave `texture.repeat = (1, 1)`. It works, but **couples geometry to texture scale**. If you globally reduce `tilesPerMetre` (for the whole scene), you must regenerate every ribbon’s UVs. With 0..1 UVs, tiling is controlled in one place (the material), which fits the `loadPbrMaterial(...).setPlaneSize(width, length)` helper.

### Gotchas

- `getSpacedPoints` distributes points evenly by arc-length, not by parameter; that is what you want so the texture does not stretch in curved areas.
- The `up × tangent` vector degenerates if the tangent is nearly vertical. Since this curve lives in XZ, that does not happen.
- Do not use `CatmullRomCurve3` with `tension > 0.5` if the waypoints are very close together; loops will bend badly.
- If the ribbon looks glued to the ground and flickers: raise `yLift` (0.005–0.02 m), or better, enable `polygonOffset` on the material.

### When not to use this pattern

- huge organic surfaces (valleys, dunes) → better use a heightmap over a subdivided plane and texture blending (see `custom-shaders.md`).
- paths that need soft edges into grass → ribbon + alpha mask in the ground shader, not an opaque ribbon.
- when the path is trivially straight → a rotated `PlaneGeometry` is more than enough.

---

## To expand

- concrete terrain blending patterns with example code
- exact KTX2 vs WebP policy per project/target
- impostors and billboarding with atlas
- realtime procedural textures and when they pay off
