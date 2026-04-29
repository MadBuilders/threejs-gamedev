---
name: threejs-gamedev
description: Build, extend and review web games with pure Three.js (no React or R3F). Covers architecture, render loop, assets, input, physics integration, rendering/RTT, performance and mobile trade-offs, audio, UI/HUD, cameras, shaders, AI/navigation, persistence, build/deploy and debugging. Use for singleplayer-first 3D/2.5D web games where control and clarity matter more than framework convenience. Not for React Three Fiber projects.
---

# Three.js Gamedev

Work with pure Three.js. Do not mix in React or R3F unless the user explicitly asks for it.

## Workflow

1. If the user wants to start a new game, settle kickoff, stack, and the first playable slice first.
2. Identify the main problem.
3. Read only the references needed (see *Context usage*).
4. Prefer maintainable patterns over demo code.
5. Treat the official docs/manual/examples/repo as the canonical base.
6. Use DeepWiki for concrete questions about the official repo structure or implementation when helpful.
7. Use semantic search in the official forum (Discourse AI) for edge cases, recurring pain points, or ecosystem-specific questions.
8. Make performance, mobile, and complexity tradeoffs explicit when they matter.

## Context usage

The skill has many references. Loading all of them in one turn is an anti-pattern.

Hard rules:
- **Maximum 3 references per turn** unless there is a clear justification.
- **Never read the advanced multiplayer block if the project is singleplayer**.
- If the user asks something cross-cutting, read the router below first and choose; if it is still unclear, ask before reading.
- If one reference points to another, do not chain reads blindly: evaluate whether the second one would truly change the answer.

## Quick router

User intent → reference to start with.

- *"I want to start a game"* → `game-kickoff-planning.md`, then `default-project-stack.md`.
- *"How should I organize the code?"* → `architecture.md`.
- *"In what order should I tackle the project?"* → `phased-game-workflow.md`.
- *"Loading models/textures/audio"* → `assets.md`, and if there is complex 3D, `gltf-pipeline.md` (includes **gltf-transform** for GLB inspection/optimization).
- *"Textures: maps, color space, tiling, compression"* → `texturing-pipeline.md` (includes **ribbon meshes over curves** for roads/rivers).
- *"The sky looks ugly / PBR materials look flat"* → `lights-shadows.md` section **IBL with HDRI**.
- *"Load GLBs/HDRIs without blocking boot"* → `assets.md` section **placeholder first, swap later** (+ correct disposal).
- *"Character animations"* → `animation-systems.md` + `animation-state-machines.md`.
- *"Move the player / follow camera"* → `character-locomotion.md` + `cameras.md`.
- *"Tank / no strafe, turn with A-D"* → `character-locomotion.md` section **tank / vehicle-lite**.
- *"Look around with the mouse without breaking camera control"* → `cameras.md` (**follow + offset**) + `input-controls.md` (**hold-to-look / pointer capture**).
- *"Minimap or radar without another render pass"* → `ui-hud.md` (**Canvas 2D**), not `render-targets.md` unless you need to see the textured world.
- *"Input (keyboard, touch, gamepad)"* → `input-controls.md`.
- *"Two virtual joysticks / mobile multi-touch"* → `input-controls.md` section **Two virtual joysticks** (key gotcha: `setPointerCapture` **per zone**, not per canvas).
- *"Mobile HUD / portrait lock / safe areas"* → `ui-hud.md` section **Real mobile HUD** (CSS-only orientation lock, `100dvh`, twin CTA for keyboard shortcuts).
- *"I need physics"* → `physics.md`.
- *"How do I do collisions without a physics engine?"* → `physics.md` section **Before adding an engine** (ladder: raycast → manual AABB/capsule → BVH → engine).
- *"Does Rapier add overhead? / should I add it now?"* → `physics.md` sections **When the switch is worth it**, **Performance** (concrete cost), and **Rapier-specific anti-patterns**.
- *"Large world / streaming / proceduralism"* → `world-generation.md`.
- *"Ugly horizon / I want real terrain relief"* → `world-generation.md` + apply **Production patterns** below (**heightfield / grid terrain**, **terrain as a system**).
- *"Empty horizon but flat gameplay"* → `world-generation.md` section **Horizon relief as silhouette** (do not add a full heightfield just for this).
- *"Hundreds of identical trees/props are slow"* → `gltf-pipeline.md` section **Migration from `clone()` to `InstancedMesh` by leaf**.
- *"Few trees/props but the game is still slow / disabling one decorative category helps a lot"* → `gpu-vs-cpu-heuristics.md` section **"Few assets" does not mean "cheap"** (real cost = tris × doubleSided × PBR × shadow × worldScale²).
- *"AI-generated asset (Meshy, Rodin, Luma, etc.), what should I review before adding it?"* → `gltf-pipeline.md` section **AI-generated GLBs** (doubleSided, 2K textures, polycount, post-download pipeline, FrontSide in loader, do not AI-remesh skinned meshes).
- *"Expensive shadows or low sharpness near the player"* → `lights-shadows.md` section **Shadow camera that follows the focus**.
- *"Should I keep receiveShadow on foliage/trees?"* → `lights-shadows.md` section **`receiveShadow` also costs per fragment** (default `false` on canopies and scattered trees).
- *"Rhythmic stutter without obviously low FPS"* → `profiling-budgets.md` (bullet about **per-frame allocations in the hot path**).
- *"I want editable source and fast runtime"* → apply **Production patterns** below and read `assets.md` / `world-generation.md` depending on the bottleneck.
- *"I want a level editor / authored map"* → `level-editor-in-browser.md` (+ **Level editors / authored worlds** below for doctrine).
- *"Clean up resources / memory leak"* → `resource-lifecycle.md`.
- *"It is slow on mobile"* → `mobile-performance.md` + `profiling-budgets.md`.
- *"Is it GPU, CPU, or stutter?"* → `gpu-vs-cpu-heuristics.md` + `frame-pacing-stutter.md`.
- *"Animations distort the mesh (root bone scale, unwanted root motion)"* → `gltf-pipeline.md` section **scale tracks**.
- *"Quality settings and scaling"* → `quality-tiers.md` + `adaptive-quality-scaling.md`.
- *"Benchmark and regressions"* → `benchmarking.md` + `stress-scenes-benchmarks.md`.
- *"Lights and shadows"* → `lights-shadows.md`.
- *"Mirrors, portals, minimaps, water"* → `render-targets.md` + `render-target-families.md` (+ `portal-*` or `minimap-fog-of-war.md` as appropriate).
- *"Transparency looks weird"* → `transparency-pitfalls.md`.
- *"Postprocessing (bloom, SSAO, etc.)"* → `postprocessing.md`.
- *"Game audio"* → `audio-systems.md`.
- *"HUD, menus, overlays"* → `ui-hud.md`.
- *"Custom shader (dissolve, water, terrain blend, etc.)"* → `custom-shaders.md`.
- *"Pathfinding / enemies / AI"* → `ai-navigation.md`.
- *"Save game, progress, settings"* → `persistence-save.md`.
- *"Build, asset compression, deploy"* → `build-deploy.md`.
- *"Visual debugging"* → `debugging.md`.
- *"Multiplayer"* → go to *Advanced block* (only on explicit demand).

## Defaults

- Pure Three.js as the base.
- `glTF` as the main 3D asset format.
- `setAnimationLoop` as the default loop.
- Separate bootstrap, render, world/systems, and gameplay when the project asks for it.
- Keep addons explicit and minimized.
- Design for clarity first, then optimize.
- **Singleplayer first** unless there is a clear multiplayer requirement.

## Production patterns

- **Editable source -> runtime artifact**: when a world/level grows, separate the file that is comfortable to edit (JSON, curves, placements) from the file that is comfortable to load in-game. Keep a fallback to source while the pipeline matures.
- **Worker-first for heavy work**: terrain, chunking, masks, clouds, or long preprocessing should not compete with input/camera/HUD on the main thread.
- **Authored data + LOD + instancing**: use authored data to decide *what* goes where; use LOD and instancing to decide *how* it is drawn cheaply. Do not mix layout with ad-hoc optimization.
- **Boot with fallbacks**: if an asset or secondary artifact is missing, the game should degrade to a simpler path instead of breaking the whole startup.
- **Heightfield / grid terrain for real relief**: if the horizon or ground truly needs shape, model terrain as height data (authored, generated, or mixed) instead of “patching” a plane with local formulas that are hard to reason about.
- **Heightfield does not imply physics**: a heightfield can serve only rendering, only placement, or rendering + colliders. Do not add Rapier “because there are hills” if the game only needs `getGroundHeight(x, z)` and maybe the terrain normal.
- **Terrain as a system, not a visual trick**: separate `terrain data` / `terrain build` / `terrain render` from the rest of `level.ts`. When relief matters, terrain, horizon, and possible colliders should come from the same mental model.
- **Rapier when the interaction asks for it**: wheels, suspension, rigid bodies resting on slopes, or continuous physical contact push toward Rapier/heightfield collider. A walking game with smooth relief can usually stay kinematic.

## Level editors / authored worlds

- **The editor writes human-readable source**: JSON or structures that are easy to inspect/diff. The editor should not write opaque formats as the main source.
- **Runtime loader separate from the editor**: `levelDefinition` / schema on one side, render/runtime on the other. That way the editor does not drag in dependencies from the whole game.
- **Bake is optional and later**: first validate that the authored workflow works. Only add an artifact/binary when size, parsing, or boot justify it.
- **Healthy fallback**: if the derived artifact is missing, load the authored source. If that fails, use an internal default so startup does not break.
- **Data before render code**: paths, placements, spawn, boundary, props, and layout tuning should live in editable data, not be buried in visual constants.
- **The editor is a route in the same bundle**: enable it with `?editor=1` and reuse bootstrap (renderer, scene, loaders) so the preview *is* the game. See `level-editor-in-browser.md` for concrete patterns: dev save endpoint, dual `TransformControls` for simultaneous translate+rotate, snap+Shift, draft in `localStorage`, extensible asset catalog, and correct helper disposal.
- **If terrain matters, it should also be authored**: for worlds with real relief, treat terrain as editable data (height grid, masks, splines, stamps, or a mix) and not as a hidden deformation in the level render code.

## Reference map

### Kickoff and project defaults
- `references/game-kickoff-planning.md` for initial questions, kickoff brief, and first playable slice.
- `references/phased-game-workflow.md` to force phases, validate mechanics before polish, and avoid scope explosion.
- `references/threejs-game-viability.md` for general viability, healthy limits, good-fit ideas, and realistic-scope inspiration.
- `references/default-project-stack.md` for the default stack, folder structure, Rapier, and singleplayer-first criteria.
- `references/default-content-sourcing.md` for opinionated sources of temporary assets, textures, and audio.
- `references/project-agents-md.md` for using `AGENTS.md` as operational memory per game.

### Core and gameplay
- `references/architecture.md` for structure, bootstrap, loop, resize, and lifecycle.
- `references/assets.md` for formats, loaders, and imports.
- `references/gltf-pipeline.md` for export, coordinated loading, compression, and instancing.
- `references/texturing-pipeline.md` for maps, color space, tiling/anisotropy, compression, and terrain blending.
- `references/animation-systems.md` for clips, mixers, actions, and blending.
- `references/animation-state-machines.md` for visual states, transitions, and one-shots.
- `references/character-locomotion.md` for player controllers, grounded state, camera, locomotion state, and variants (**tank controls** when strafe competes with another mechanic).
- `references/cameras.md` for follow cameras, spring-damped, orbital, cinematic, and collision-aware.
- `references/input-controls.md` for input abstraction, keyboard, touch, gamepad, and raycasting.
- `references/physics.md` for physics engine integration and responsibility boundaries.
- `references/world-generation.md` for streaming, chunking, and procedural content.
- `references/level-editor-in-browser.md` for putting a level editor inside the same app (dev save endpoint, dual `TransformControls`, snap, draft in `localStorage`, asset catalog, disposal gotchas).
- `references/ai-navigation.md` for pathfinding, nav meshes, steering, and simple behavior.
- `references/resource-lifecycle.md` for ownership, cleanup, and `dispose()`.

### Presentation and UX
- `references/audio-systems.md` for buses, spatial audio, loading, and voice pools.
- `references/ui-hud.md` for HUD, menus, DOM vs canvas overlays, and healthy coupling.
- `references/persistence-save.md` for saving game state, progress, and settings.

### Performance and validation
- `references/mobile-performance.md` for budgets and cost reduction.
- `references/profiling-budgets.md` for frame time, draw calls, and real budgets.
- `references/gpu-vs-cpu-heuristics.md` for distinguishing visual, logic, mixed, or stutter bottlenecks.
- `references/frame-pacing-stutter.md` for spikes, warmup, and smooth activation.
- `references/quality-tiers.md` for coherent presets by device.
- `references/adaptive-quality-scaling.md` for hysteresis, cooldown, and `renderScale`.
- `references/stress-scenes-benchmarks.md` for internal benches and stress scenes.
- `references/benchmarking.md` for reproducible runs, diffs, thresholds, and final classification (unified reporting + diffs + thresholds).

### Rendering, RTT, and lighting
- `references/render-targets.md` for RTT as a subsystem, resolution, frequency, and lifecycle.
- `references/render-target-families.md` for mirrors, refractors, portals, and minimaps.
- `references/portal-recursion.md` for depth, per-level resolution, and fallbacks.
- `references/portal-masking-stencil-scissor.md` for area clipping, stencil, and overdraw.
- `references/minimap-fog-of-war.md` for tactical minimaps, visibility, and explored state.
- `references/fog-mask-blending.md` for fog masks and blending.
- `references/transparency-pitfalls.md` for sorting, depth, alpha test, and healthy decisions with transparent materials.
- `references/lights-shadows.md` for lighting strategy and shadow maps.
- `references/postprocessing.md` for effect chains, resize, and usage criteria.
- `references/custom-shaders.md` for `ShaderMaterial`, `onBeforeCompile`, common patterns, and anti-patterns.

### Debug and build
- `references/debugging.md` for helpers and visual inspection.
- `references/build-deploy.md` for Vite build, asset compression, cache busting, and deploy.

### Advanced block (only on explicit demand)
Do not load these references by default. Enter here only if the user declares multiplayer as core to the game, or if they are adding it to an existing singleplayer project.

- `references/multiplayer.md` for base network architecture, snapshots, interest management, and the **concrete recommended stack (Colyseus)** with its 0.17 gotchas.
- `references/multiplayer-consistency-models.md` for rollback, lockstep, and hit validation.
- `references/server-rewind-weapons.md` for per-weapon rewind or lag compensation.
- `references/anti-cheat-anomalies.md` for telemetry, suspicion scoring, and mitigations.

Healthy default for casual / cooperative / lightweight competitive games: start with `multiplayer.md` and propose Colyseus with a monorepo (`client/` + `server/`). Jump to the other three only if the genre justifies it.

## Judgment rules

- Do not copy official documentation into the skill.
- Do not present demo code as production architecture.
- Mark what is core, what is addon, and what is project doctrine.
- Recommend external tools only when they add a clear pipeline.
- If a decision affects mobile or performance, make the tradeoff explicit.
- If a reference declares an anti-pattern, do not contradict it without justifying why this case is an exception.

## Base sources

- `threejs.org/docs`
- `threejs.org/manual`
- `threejs.org/examples`
- `github.com/mrdoob/three.js`
- DeepWiki on the official repo as targeted help
- semantic search in the official forum (`/discourse-ai/embeddings/semantic-search.json`) as targeted help for concrete problems

## Current state

**v1.12**. Captured learnings from porting a singleplayer Three.js game to mobile with a two-virtual-joystick scheme:

- `input-controls.md`: new section **Two virtual joysticks (reusable pattern)**. Concrete time-saving gotchas: **`setPointerCapture(e.pointerId)` per zone, not per canvas** (without that, iOS Safari routes the second finger's `pointermove` to whoever has the most recent capture and the diagonal breaks), **dynamic center** in the stick base, dead zone ~0.12 + 60 px radius as defaults, **touch as an override over the same `axes` struct** (not a parallel pipeline: gameplay reads one struct and desktop code does not need `if (isTouch)`), **hold-to-look and two sticks are mutually exclusive on mobile** (you must *remove* `pointerdown` from the canvas, not merely stop drawing it), `touch-action: none` + `user-select: none` on the canvas but not on `body`, and **device class once at boot** with `matchMedia('(pointer: coarse)')` → `body.is-touch`. Removed `virtual joystick` from *Pending expansion*.
- `ui-hud.md`: new section **Real mobile HUD (landscape first)** with patterns that are not gamedev-obvious but are mobile-obvious: `100dvh` on critical containers because of iOS Safari URL bar collapse, `env(safe-area-inset-*)` also on outer CTA padding (not only on `body`), **CSS-only orientation lock** (make explicit that the `ScreenOrientation API` should not be called: uneven support, requires fullscreen), mandatory touch twin for any keyboard shortcut in game-over / paused, and the same `body.is-touch` as the shared switch with the input system.
- `SKILL.md`: two new router entries (*"Two virtual joysticks / mobile multi-touch"* and *"Mobile HUD / portrait lock / safe areas"*) so they land directly in the new sections.

**v1.11**. Strengthened collision/physics doctrine based on common kickoff questions ("how does collision work in Three.js?", "does Rapier add overhead if I add it?"):

- `physics.md`: new block at the beginning, **Before adding an engine**, with the explicit statement *"Three.js has no collision detection"* and the **four-strategy ladder** (point raycast → manual bounding volumes → BVH over static mesh → physics engine). It explicitly permits staying on step 2 (manual XZ or AABB collision) for most casual/walking/light-multiplayer games.
- `physics.md`: new section **The collider is never the visual asset** as a hard rule. The GLB is rendered; you declare the collider yourself (radius, capsule, AABB). Complements the warning in `gltf-pipeline.md` about AI GLBs with `doubleSided: true` and 2–3M tris.
- `physics.md`: new section **When the switch to a physics engine is worth it** with an operational checklist (wheels/suspension, stacking, slopes with continuous contact, physical projectiles, joints/constraints, emergent mechanics). Replaces the generic "when to use full physics" with concrete signals. Ends with the corollary *"Rapier when the interaction asks for it"*.
- `physics.md`: expanded **Performance** section with the **concrete cost of Rapier** (1–1.5 MB compat bundle, async init, step cost by body type, mandatory fixed timestep, sync without allocations, cacheable queries) and orienting **mental numbers** (<1 ms with dozens of kinematics, 2–5 ms with 500+ dynamics, 10–20 ms with a trimesh for the entire level).
- `physics.md`: new section **Rapier-specific anti-patterns** with the 8 typical footguns (world trimesh, dynamic by default, broken sleep, create/destroy in hot path, variable delta in step, per-NPC raycasts, sync with allocations, collider = GLB).
- `SKILL.md`: two new router entries (*"How do I do collisions without a physics engine?"* and *"Does Rapier add overhead? / should I add it now?"*) so these questions land directly in the new `physics.md` sections instead of the generic *"I need physics"* entry.

**v1.10**. Captured learnings from an optimization session focused on AI-generated assets in a real 3D game:

- `gltf-pipeline.md`: new section **AI-generated GLBs (Meshy and similar)** with their silent defaults (`doubleSided: true` on opaque materials, 2K textures per PBR slot, 0.5M–3M triangle polycount, no compression), the canonical post-download pipeline (`resize` → `webp` → `meshopt`), the rule to **force `FrontSide` in the loader** instead of correcting the GLB (robustness against future re-exports), the warning to **not pass skinned meshes through AI-remesh** (breaks weights/skeleton/animations), and the practice of **variant scale as authored data, not baked in the GLB** so the mesh can be regenerated without retuning the scene.
- `gpu-vs-cpu-heuristics.md`: new section **"Few assets" does not mean "cheap"** with the mental formula for real GPU cost per asset (`tris × instances × fragments × doubleSided × PBR samples × shadow samples × worldScale²`) and the explicit warning that **`InstancedMesh` collapses draw calls but does not lower per-instance cost**. New GPU-bound symptom: disabling an entire decorative category wins a lot even when there are only 20 instances.
- `lights-shadows.md`: new section **`receiveShadow` also costs per fragment** — not only `castShadow`. In dense foliage and canopies above the player, `receiveShadow = false` is almost always the right answer. Recommendation to expose the flag in loader factories so trees and props can share a loader but diverge in shadow policy.
- `SKILL.md`: three new router entries (AI assets, few assets tanking performance, `receiveShadow` in foliage).

**v1.9**. Captured learnings from an optimization pass on a 3D singleplayer game already in production:

- `world-generation.md`: new section **Horizon relief as silhouette** for the case “gameplay is flat but the horizon looks empty.” Reusable recipe based on **radial mask · angular pattern · fbm detail**, with tradeoffs versus authored heightfields and an explicit warning that it is render-only (do not touch colliders / navmesh with this).
- `gltf-pipeline.md` (Case 3): new section **Migration from `clone()` to `InstancedMesh` by leaf** with the reusable template for flattening leaf meshes on load and exposing a factory `createInstancedMeshes(placements)` parallel to the existing `instance()`. Includes rules about grouping by variant, keeping `instance()` for interactive cases, and what happens with obstacles/collision.
- `lights-shadows.md`: new section **Shadow camera that follows the focus** (sun position + `sun.target` moving with the player while preserving the original offset) as an alternative to a huge static frustum. Allows reducing `mapSize` and combining with `PCFShadowMap` without losing sharpness nearby.
- `assets.md`: new section **Parallel prefetch for prop collections** with the `Promise.all(uniqueKinds.map(getModel))` pattern when the loader already caches by kind; removes hidden serialization of the first prop of each type. Note about limited pools if unique kinds number in the hundreds.
- `ui-hud.md` (minimap): extra bullet about **cull by distance before pixel math** (`dx² + dz² > radius² · 2`) when the obstacle list grows into the hundreds.
- `profiling-budgets.md`: the CPU-bound bullet about "avoidable JS work" expands to **per-frame allocations in the hot path**, with the recipe of module-level scratch buffers and the gotcha of not reassigning arrays shared with `result.xxx` (use `length = 0`).
- `SKILL.md`: five new router entries for each of the previous patterns.

**v1.8**. More explicit terrain/worldbuilding for games with real relief:

- `SKILL.md`: quick router with a new entry for *ugly horizon / I want real terrain relief*.
- `SKILL.md`: in **Production patterns**, two new heuristics added: **heightfield / grid terrain** and **terrain as a system**.
- `SKILL.md`: clarified that **heightfield does not imply Rapier** and that Rapier enters mostly when there are wheels/suspension or continuous physical contact.
- `SKILL.md`: in **Level editors / authored worlds**, fixed that if terrain matters, it should also be authored as editable data and not live as an opaque deformation in render code.

**v1.7**. Captured learnings from building a real level editor inside the game bundle:

- New reference `level-editor-in-browser.md` with proven patterns: editor as subroute `?editor=1` that reuses bootstrap, save through dev endpoint `POST /dev/level` with `NODE_ENV` guard, `localStorage` draft with silent restore, **dual `TransformControls`** on the same object (simultaneous translate XZ + rotate Y) to avoid the move/rotate toggle, 0.5m / 15° snap with Shift for free movement, centralized asset catalog for extensibility without touching the editor, schema backward compatibility with optional fields + `resolveXxx` helpers, and the classic `group.children` trap where the group is not emptied by calling only `dispose()` (ghost helpers).
- `SKILL.md`: router and **Level editors / authored worlds** block point to the new reference. Added doctrine that **the editor is a route in the same bundle**, not a separate app.

**v1.6**. More explicit authoring/editor pipeline:

- `SKILL.md`: new router entry for **level editor / authored map**.
- New **Level editors / authored worlds** block to fix the reusable pattern: human-readable source, separate loader, optional bake, fallback to source, and layout saved as data.

**v1.5**. Production patterns reinforced from shipped web games:

- `SKILL.md`: new **Production patterns** block to remember four heuristics that often arrive late but should be modeled early: **editable source -> runtime artifact**, **worker-first heavy subsystems**, **authored data + LOD + instancing**, and **boot with fallbacks**.
- Quick router: added an explicit entry for when the problem is less "how do I render this" and more "how do I move from data/editing to runtime without breaking the project".

**v1.4**. Adjustments after continuing to iterate on the same real project into a more mature phase:

- `project-agents-md.md`: reinforced that `AGENTS.md` should be **refreshed when the project changes phase** and that it is worth opening satellite documents (`MULTIPLAYER.md`, etc.) when a subsystem gains its own roadmap. Prevents operational memory from freezing at V0.
- `default-project-stack.md`: added healthy evolution pattern to **monorepo `client/` + `server/` + `shared/`** when multiplayer stops being hypothetical, and a note about authoring/data-driven layout (`public/levels/` + explicit definitions) so playable content is not buried in scattered constants.
- `multiplayer.md`: new concrete rules about **validating score/progress against state the server already knows** (not against claim payloads) and about **not blocking rounds on external persistence**; leaderboard persistence in background + timeout as the healthy default.

**v1.3**. Added learnings from a real multiplayer project (pure Three.js client + Colyseus server in a monorepo):

- `multiplayer.md`: new section **Concrete recommended stack: Colyseus** with when to choose it, when not to, and the **0.17 gotchas** that cost time (`MapSchema` not iterable, `getStateCallbacks` replaces direct `onAdd`/`onRemove`, late state hydration, `useDefineForClassFields: false`, `@types/express` for Express 5). Also: integration pattern with non-blocking connection, a single `MultiplayerHandle` as the isolation layer, deterministic server-side visual identity (color hue from a fixed palette), and a multi-client headless smoke test.
- `animation-systems.md`: section **Concrete gotchas when cloning SkinnedMesh** (`Object3D.clone` is not enough, named exports from `SkeletonUtils.js`, materials and geometries shared by the clone, ownership rule in `dispose`) + **source + instance** pattern (`loadCharacterSource` / `createCharacterInstance`) to reuse GLBs between local player, NPCs, and remotes without refetch or double parse. Anti-patterns extended.

**v1.2**. Generic patterns proven in web production:
- `cameras.md`: **follow from behind + yaw/pitch offset with decay** (visuals decoupled from the movement frame when another mechanic fixes the reference).
- `input-controls.md`: **hold-to-look** with `pointerdown` + `setPointerCapture` as an alternative to pointer lock.
- `character-locomotion.md`: **tank / vehicle-lite** (W/S axis, A/D turn; facing as state, not derived from velocity).
- `ui-hud.md`: **minimap/radar with Canvas 2D**, player-up.
- `gltf-pipeline.md`: **scale tracks** in clips that distort retargeted rigs + **gltf-transform (CLI)** section.
- `texturing-pipeline.md` new: maps, color space, tiling, compression, and **ribbon meshes over curves** for roads/rivers.
- `lights-shadows.md`: **IBL with HDRI** (`PMREMGenerator` as `scene.environment` + `scene.background`).
- `assets.md`: **placeholder first, swap later** and `dispose()` recipes when replacing material/texture/render-target.

Router updated. The advanced multiplayer block now has a concrete default (Colyseus) in addition to the general doctrine; it is still loaded only on demand.
