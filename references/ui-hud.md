# UI and HUD

## Objective
Decide how to structure HUD, menus, and overlays in a pure Three.js game without adding unnecessary frameworks or hiding game logic inside the UI layer.

## Main rule
**UI reflects game state; it does not dictate it.**
Gameplay publishes events/state. UI subscribes and renders. Never call gameplay logic directly from a menu `onClick`.

## Base decision: DOM vs 3D canvas
By default:
- **DOM/CSS** over the canvas for HUD, menus, dialogs, options.
- **3D canvas** (sprites, textured meshes) only for diegetic UI (world markers, health bars over characters, integrated minimap).

Reasons:
- DOM wins for accessibility, layout, text, and localization.
- DOM is cheap and does not compete for the game's draw calls.
- 3D canvas is appropriate when the UI is part of the world (HMD, cockpit, physical panel).

Intermediate cases (very artistic HUD with rich animations): DOM with CSS/SVG or separate WebGL, but do not put it in the main render unless necessary.

## Minimum stack
- Plain HTML in `index.html` for the shell (canvas + overlay container).
- CSS for layout.
- No UI framework by default. If the game has many screens, consider something very lightweight (do not mount React just for menus).
- Event bus or simple game store as the source of truth for UI.

## Typical shell structure

```html
<div id="app">
  <canvas id="game"></canvas>
  <div id="hud"></div>
  <div id="menus"></div>
  <div id="toasts"></div>
</div>
```

Layers:
- `#game`: full-screen Three.js canvas.
- `#hud`: always-visible overlay, `pointer-events: none` except on interactive elements.
- `#menus`: modal screens (pause, settings, game over).
- `#toasts`: ephemeral feedback.

## Pointer events and input
- The canvas receives gameplay input.
- HUD should be `pointer-events: none` by default so it does not steal clicks. Only specific buttons reactivate `pointer-events: auto`.
- If a modal UI is open, gameplay should ignore input while the UI consumes it. Ideally, the input system knows an `inputContext` (gameplay, menu, dialog) and routes accordingly.

## Healthy gameplay ↔ UI coupling
Pattern:
1. Gameplay exposes state (`player.health`, `run.score`, `world.currentObjective`).
2. Gameplay emits domain events (`onObjectiveReached`, `onRunFailed`, `onPause`).
3. UI observes state or subscribes to events and renders.
4. UI never mutates gameplay state directly. It calls well-defined *commands* (`pause()`, `requestRestart()`, `setSettingsVolume()`).

This allows HUDs to change without touching gameplay and gameplay to be tested without UI.

## Recommended data flow
For simple state: a `GameState` object + callbacks.
For more game: a small store with subscription (Redux is not needed; a `Map<string, Set<Listener>>` is enough).

Avoid:
- querying `scene.getObjectByName(...)` from UI
- letting the HUD have its own truth for `playerHealth` that differs from gameplay

## Resize and pixel ratio
- HUD scales with the viewport. Avoid absolute px sizes for critical elements; use `clamp()` or CSS variables derived from the viewport.
- Account for safe areas on mobile (`env(safe-area-inset-*)`).
- HUD must not depend on the renderer's `renderScale`: that only affects the canvas.

## HUD on real mobile (landscape first)

When the game is playable on phone/tablet, the desktop HUD usually does not port directly. Concrete patterns that are not gamedev-obvious but are mobile-obvious:

- **`100dvh`** on critical containers (overlays, modals, full-screen panels). `100vh` on iOS Safari can be too short when the URL bar is visible or too tall when it collapses; `dvh` recalculates against the real viewport and stops dragging cuts through the HUD.
- **`env(safe-area-inset-*)` also on the outer padding** of HUD and CTAs, not only on `body`. Corner buttons need explicit respect for the notch/home indicator; otherwise they get covered or sit outside the touchable area on iPhone.
- **Orientation lock is CSS-only**. Do not call the `ScreenOrientation API`: browser support is uneven and several browsers require fullscreen. An overlay "Rotate to play" revealed by `@media (pointer: coarse) and (orientation: portrait)`, with `visibility: hidden` on `#hud` behind it, is enough. Zero cost and works everywhere.
- **Every keyboard-dependent CTA needs a visible touch twin**. If desktop says "Press R to restart", mobile HUD needs a real `Play again` button routed to the same command. Hide one or the other with `body.is-touch` (or `@media (pointer: coarse)`) so desktop and mobile do not overlap; never leave the player staring at a shortcut their device cannot press.
- **Device class once at boot**: `matchMedia('(pointer: coarse)')` → `is-touch` class on `<body>`. CSS (compact layout, smaller minimap, panels with `overflow-y: auto`, visible joysticks) and JS (input system, UI wiring) share the same switch. Same pattern documented in `input-controls.md` — one detection, shared.

## Diegetic 3D HUD
When HUD lives in the world:
- Use `Sprite` for markers that always face the camera.
- Health bars over characters: textured plane with appropriate `depthTest`.
- Avoid real 3D text (geometry cost) unless it is the style. Prefer pre-rendered text textures or atlases.
- For lots of dynamic text, consider `troika-three-text` (external addon) with judgment.

## Menus and screens
- Menu state as a simple machine: `boot → mainMenu → gameplay → paused → gameOver → mainMenu`.
- Each screen is a DOM component/node that is shown/hidden.
- Short CSS transitions, without blocking the game loop.
- Real pause: the game loop keeps rendering but stops simulation time (`dt = 0`). The world stays still while the scene remains visually alive.

## Accessibility and localization
- Use semantic elements (`button`, `dialog`, ARIA roles) in DOM.
- Minimum touch sizes on mobile (~44px).
- Centralize text in an i18n module, even if it is just a `Record<string, string>` to start.
- Do not hardcode text across loose templates.

## Performance
- Calm DOM does not compete with rendering. Heavy CSS animations (large shadows, blurs) can cost, especially on mobile.
- Avoid per-frame reflows (touching `layout` every update). Batch changes or write to CSS variables.
- Canvas 2D for highly dynamic HUDs with many elements can be cheaper than DOM.

## Cheap minimap / radar (Canvas 2D)

For **orientation** (goal, spawn, obstacles), you do not need a second Three.js render pass or RTT of the scene.

Pattern:
- A 2D `<canvas>` in the DOM overlay (same logical resolution, `devicePixelRatio` in the backing store if you want crispness).
- Every frame: `clearRect`, draw points/rectangles in **world coordinates → pixels** with scale `metersPerPixel = radiusMeters / (canvasSize/2)`.
- **Player-up radar**: `ctx.translate(cx, cy); ctx.rotate(facing − π)` (or whatever convention fits your `forward = (sin f, cos f)`), draw goal/spawn/obstacles **under** that rotation; the player icon (triangle) and a fixed cardinal mark **above**, without rotation, so “up = character forward”.
- Goal out of range: project it to the circle edge (clamp by magnitude) and draw an **arrow pointing radially outward** (rotated so its apex matches the goal direction).
- **Cull by distance before pixel math**: when obstacles grow (hundreds of trees/props in open worlds), iterating the full list every frame while most are off-radar wastes Canvas 2D CPU. Early-out with `dx² + dz² > radius² · 2` (√2 factor so boxes whose center is off-radar but whose half-extent still enters the disk are not clipped) before calculating pixel coordinates or calling `fillRect`.

Advantage over `WebGLRenderTarget` + overhead camera: almost zero cost (~dozens of 2D primitives per frame), with no second frustum or depth clearing. See also `render-target-families.md` when you really need **the real view** textured ("photographic" map, fog of war, etc.).

## Debug UI
Separate from the final HUD, activatable by key or query param:
- FPS and frame time
- counters (`renderer.info`)
- player state
- quick toggles

Never leave debug UI loaded in production without a flag.

## Anti-patterns
- mounting React/Vue/Svelte for a HUD with 4 indicators
- UI reading from the scene graph instead of game state
- `onClick` doing direct gameplay (move character, shoot)
- HUD with z-index fighting the canvas and solved with `!important`
- real 3D text for every HUD number
- not distinguishing game pause from visual pause
- blocking global input without a context machine
- localization hand-written in multiple places

## Strong recommendation
From day one:
- HTML shell with `hud`, `menus`, `toasts` layers
- `pointer-events: none` on HUD by default
- simple store or event bus as the source of truth
- screen state machine
- texts by key, even if there is only one language at first

## Related references
- `architecture.md`
- `input-controls.md`
- `audio-systems.md`
- `persistence-save.md`
