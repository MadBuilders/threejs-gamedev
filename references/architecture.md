# Architecture

## Goal
Define a reasonable foundation for pure Three.js games that does not collapse into a single file and can grow without mixing responsibilities too early.

## Main rule
Separate, at minimum, these layers:

1. **Bootstrap**
   - create renderer
   - create scene and main camera
   - mount canvas
   - register resize
   - start loop

2. **World / Scene setup**
   - lights
   - environment
   - floor or base geometry
   - persistent world objects

3. **Systems**
   - input
   - camera
   - animation
   - audio
   - networking bridge if there is multiplayer
   - physics bridge
   - spawners
   - UI bridge if applicable

4. **Gameplay**
   - rules
   - score
   - match/game states
   - objectives
   - progression

5. **Entities / Actors**
   - player
   - enemies
   - pickups
   - obstacles
   - interactive props

## Suggested minimum structure

```text
src/
  main.js
  app/
    bootstrap.js
    loop.js
    resize.js
  world/
    createScene.js
    createLights.js
    createEnvironment.js
  systems/
    inputSystem.js
    cameraSystem.js
    animationSystem.js
  gameplay/
    gameState.js
    rules.js
  entities/
    player.js
    pickup.js
```

Adapt granularity to the real size of the project. Do not fragment for show.

## Loop
Use `renderer.setAnimationLoop()` by default.

Separate inside the loop:
- input read
- systems update
- gameplay update
- final visual sync
- render

Avoid putting arbitrary logic directly in DOM callbacks or inside `render()`.

`setAnimationLoop()` works well as a game foundation and also keeps a natural path open if XR arrives later. Still, do not confuse the render loop with permission to update everything without control. If part of the system can live outside the critical frame, that is better.

## Resize and lifecycle
Centralize:
- current width/height
- aspect ratio
- `camera.updateProjectionMatrix()`
- `renderer.setSize()`
- if needed, `renderer.setPixelRatio()` with reasonable limits

## Useful conventions
- Keep explicit references to shared systems.
- Avoid uncontrolled global singletons.
- Pass important dependencies through composition when reasonable.
- Do not couple game logic to one concrete camera more than necessary.
- Do not mix asset loading with gameplay rules.
- If there is multiplayer, do not use the scene graph as the source of truth for shared state.

## Core vs addons
Reviewing the repo and docs leaves a useful rule: clearly distinguish what is **core** Three.js and what comes through **addons/examples**.

- core: scene, camera, renderer, materials, geometries, math, `Object3D`, `Raycaster`, etc.
- addons: specific loaders, controls, postprocessing, and many utilities from examples

In the game architecture, addons should enter as explicit dependencies of concrete systems, not be scattered around as if they were engine fundamentals.

## Recommended bootstrap (TS)

### Bad example
Everything stuck in `main.ts`, input and gameplay mixed, ad-hoc resize and loop:

```ts
const canvas = document.querySelector('canvas')!;
const renderer = new THREE.WebGLRenderer({ canvas });
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight);
renderer.setSize(innerWidth, innerHeight);

const player = new THREE.Mesh(/* ... */);
scene.add(player);

addEventListener('keydown', (e) => {
  if (e.key === 'ArrowUp') player.position.z -= 0.1;
});

addEventListener('resize', () => {
  renderer.setSize(innerWidth, innerHeight);
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
});

function tick() {
  renderer.render(scene, camera);
  requestAnimationFrame(tick);
}
tick();
```

Problems: DOM input touches gameplay directly, resize does three things, there is no `dt`, there is no `dispose`, and the camera and player have no systems behind them.

### Good example
Explicit layers, loop with `dt`, centralized resize, abstracted input:

```ts
const renderer = createRenderer(canvas);
const scene = createScene();
const camera = createMainCamera();
const world = createWorld(scene);
const input = createInputSystem();
const player = createPlayer({ scene, input });
const cameraRig = createCameraRig(camera, player);

const resize = createResize(renderer, camera);
resize.install();

const clock = new THREE.Clock();

renderer.setAnimationLoop(() => {
  const dt = Math.min(clock.getDelta(), 0.1);
  input.update(dt);
  player.update(dt);
  cameraRig.update(dt);
  world.update(dt);
  renderer.render(scene, camera);
});
```

Keys:
- each factory lives in its own module (`app/bootstrap`, `systems/input`, `entities/player`, `world/...`).
- clamp `dt` to avoid huge steps when returning from a suspended tab.
- input is a system, not a loose listener.
- resize is centralized, with cleanup if needed.
- the camera is a rig that consumes state, not a child of the mesh (see `cameras.md`).

## Initial anti-patterns
- giant `main.ts` with everything mixed together
- DOM input triggering gameplay directly everywhere
- assets loaded from any file without coordination
- camera, player, and rules glued into a single class
- custom `requestAnimationFrame` instead of `setAnimationLoop`
- update loop without `dt` (everything coupled to frame rate)
- not clamping `dt`: an inactive tab returns a huge delta and breaks physics
- optimizing too early without measuring the real cost

## Related references
- `phased-game-workflow.md`
- `default-project-stack.md`
- `resource-lifecycle.md`
- `cameras.md`
- `input-controls.md`
