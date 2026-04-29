# Lights and Shadows

## Goal
Make lighting and shadow decisions in Three.js with a web-game mindset, not as an isolated demo.

## Main rule
Shadows are expensive. Very expensive if they are allowed to grow without control.

The manual makes this crystal clear: every light that generates shadows forces the scene to be rendered again from that light’s point of view. With several lights, the cost multiplies very quickly.

## Recommended default
- start with simple lighting
- use a few important lights
- if dynamic shadow is needed, prefer **one main directional light** over several shadow-casting lights
- treat shadows as a limited budget, not as a universal default

## Shadow strategy by level

### Level 0, no dynamic shadows
Use:
- simple lighting
- AO or light hints in assets
- material contrast
- visual composition of the scene

### Level 1, fake shadows
Highly recommended in many stylized or mobile games.

Classic pattern:
- simple plane or decal
- soft grayscale shadow texture
- `MeshBasicMaterial`
- `transparent: true`
- `depthWrite: false`
- place slightly above the ground to avoid z-fighting

This is extremely cheap and often sells the effect well enough.

### Level 2, real shadow maps
Use them when they truly improve game readability or fantasy.

Rules:
- enable `renderer.shadowMap.enabled` only when appropriate
- set `castShadow` and `receiveShadow` deliberately, object by object
- tune the light’s shadow camera; do not leave it absurdly broad by default
- measure cost in real scenes

## Lights
Do not add lights just because.

Useful questions:
- what does this light add to game readability?
- can we achieve almost the same thing with one fewer light?
- does the visual style need realism or clarity?

## Shadow camera
The manual stresses something important: if shadows are missing or weird, the problem is often not “Three.js” but the region covered by the shadow camera.

Practical rule:
- visualize it with `CameraHelper`
- tune top, bottom, left, right, near, and far to the real useful area
- avoid giant boxes if the action happens in a small area

### Shadow camera that follows the focus
For a game with a **large playable area** and shadows that matter near the player, the temptation is to enlarge the shadow camera frustum until it covers the entire map. That is expensive twice: it lowers effective resolution (each shadow-map texel covers more world) and forces you to raise `mapSize` to compensate.

Cheaper pattern: **keep a small, tight frustum and move it with the player** (or camera focus).

Recipe:
- At boot, store the original sun offset as `SUN_OFFSET = sun.position.clone()`.
- `scene.add(sun.target)`: the `DirectionalLight` points from `position` to `target.position`, so the target must be in the scene for the matrix to update.
- Every frame, after moving the player:
  ```ts
  sun.position.set(player.x + SUN_OFFSET.x, SUN_OFFSET.y, player.z + SUN_OFFSET.z);
  sun.target.position.set(player.x, 0, player.z);
  sun.target.updateMatrixWorld(true);
  ```
- With the frustum parked over the player, you can use half-extents around “what the third-person camera sees nearby” (for example ~20–25 m) and a lower `mapSize` (`512²` or `768²`) without losing sharpness.

Tradeoffs:
- the sun direction stays constant (it only moves, not rotates), so lighting remains stable as the player advances.
- shadows far from the player disappear; for a third-person single-player game, that is usually what you want. If gameplay depends on distant shadows (stealth, puzzles), this pattern does not fit.
- it combines well with `PCFShadowMap` (cheaper) instead of `PCFSoftShadowMap`; with a tight frustum, the extra blur is usually unnecessary.
- if you lower `mapSize` a lot, increase negative `shadow.bias` slightly (`-0.0005` is a healthy starting point) to avoid acne on the ground.

### `receiveShadow` also costs per fragment

It is easy to think only of `castShadow` as the expensive lever (it triggers the extra shadow pass). But `receiveShadow = true` adds **a shadow-map sample for every fragment of the mesh** in the main render — a PCF lookup that is noticeable when fragment density is high.

Where it matters:
- **dense foliage**: tree crowns, grass, leaves. Tens of thousands of visible fragments per frame, and the canopy is *above* the player → receiving shadow there adds (almost) nothing visually.
- **surfaces that never receive a realistic shadow**: skies, backgrounds, internal building faces the main light cannot reach.

Healthy pattern:
- `castShadow` for gameplay (projecting the player, enemies, interactive props).
- `receiveShadow` only if **something will actually cast a shadow onto it**. Not as a universal default.
- For scattered trees: `receiveShadow = false` is almost always correct. The ground needs it; the crown does not.

Expose the flag as an option in load factories (`loadTreeModel({ receiveShadow: false })`) so trees and props can share the loader while diverging in shadow policy: props resting on the ground do want the player’s shadow to fall on them; scattered trees do not.

## Helpers and debug
Useful when working with shadows:
- `CameraHelper` on the shadow camera
- tools to view the shadow map if the case gets complicated
- HUD or debug toggles to compare with and without shadows

## Anti-patterns
- several shadow-casting lights by default
- `castShadow` and `receiveShadow` enabled on everything
- complex shadow maps on mobile without measuring
- not tuning the shadow camera
- chasing perfect shadows when the game style does not need them

## Strong recommendation
For many web games, the winning combination is:
- one well-chosen main light
- fake shadows for secondary elements
- or directly, very selective shadows only where they truly help

## IBL with HDRI (image-based lighting)
Disproportionately useful technique for PBR scenes: load an equirectangular HDRI, pass it through `PMREMGenerator`, and assign it to both `scene.background` and `scene.environment`. With one call you get:
- **skybox** behind everything (solves “the sky looks ugly”),
- coherent **reflection + diffuse ambient** for all PBR materials, without adding extra lights.

Minimal recipe:
```ts
// Three r168+ replaced RGBELoader with HDRLoader. Equivalent API.
const hdr = await new HDRLoader().loadAsync(url);
const pmrem = new THREE.PMREMGenerator(renderer);
const envRT = pmrem.fromEquirectangular(hdr);
hdr.dispose();
pmrem.dispose();
scene.background = envRT.texture;
scene.environment = envRT.texture;
```

When it pays off:
- any outdoor scene with PBR materials
- you want characters, props, and metals to light “well” without fighting 3–4 point lights
- the camera sees real sky and you do not want a painted background

Gotchas:
- if you already had `HemisphereLight` or strong ambient lights, **lower them when you add an env map**, or everything becomes double-lit and contrast washes out.
- the `DirectionalLight` representing the sun is still needed if you want shadows (the env map alone does not cast them).
- the HDRI sun direction ≈ your `DirectionalLight` direction, or it will be obvious.
- Poly Haven (CC0) is the reasonable default; 1K is almost always enough for the background, 2K if the sky is very present on screen.
- decompressing an HDR to PMREM has a cost; do it once at boot, not per frame.
- dispose the PMREM `RenderTarget` if the scene is destroyed (what `fromEquirectangular` returns is a `WebGLRenderTarget`).

Anti-pattern: `new RGBELoader().load(...)` and assigning it directly to `scene.environment` without PMREM. It appears to work, but reflections get artifacts and ambient lighting is poorly prefiltered.

`backgroundIntensity` (Three r155+) lets you lower the visible sky brightness without touching IBL intensity on materials, useful when the HDRI is too bright as a background but works as IBL.

## To expand
- `shadowMap` types
- directional / spot / point comparison in real cost
- shadow policy by quality preset
- hybrid techniques with lightmaps or AO
