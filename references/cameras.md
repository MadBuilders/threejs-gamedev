# Cameras

## Objective
Define useful camera strategies for Three.js games: follow cameras, orbital, top-down, cinematic, collision-aware, without coupling the camera to the rest of the game more than necessary.

## Main rule
**The camera is a system, not a child of the player.**
It should consume state (position, velocity, input intentions) and produce a transform. It does not modify gameplay and does not live glued to the character graph.

## Choose the camera type before the code
Mandatory kickoff question (see `game-kickoff-planning.md`): first-person, third-person, top-down, side-view, free? Each implies different tradeoffs in input, collisions, rendering, and UI.

## Main patterns

### 1. Follow camera (third-person)
- target offset in character space (behind and above).
- smooth interpolation (spring-damped or `damp()` per axis).
- rotation controlled by input (mouse, right stick, touch).
- look at the point of interest: typically `target + lookOffset`, not the character pivot.

### 2. Orbital
- for vehicles, puzzles, photo modes, editors.
- based on `OrbitControls` (addon) or a custom implementation if input is mixed.
- limit polar angle and distance to avoid broken angles.

### 3. Top-down / isometric
- orthographic camera or perspective camera with low FOV.
- target follows the player on the horizontal plane with smooth lag.
- zoom as another controllable axis.
- watch shadows: orthographic cameras need `shadow.camera` adjusted accordingly.

### 4. First-person
- camera is a logical child of the character, but not literally of the mesh.
- look controlled by input with pitch clamp.
- separate body position from head position to allow head bob and smoothing.

### 5. Cinematic / scripted
- keyframe timeline with position, target, and FOV.
- interpolation with easing.
- in short cinematics, freeze gameplay input; in longer ones, consider skip.

## Damping and spring
- Never bind the camera directly to the character (`camera.position.copy(player.position)`).
- Prefer interpolation by delta time:
  - `damp(current, target, lambda, dt)` where `lambda` controls stiffness.
- Use different `lambda` values for position and rotation; rotation is usually faster.
- Do not depend on `frameRate`: if using `lerp` with a fixed factor, tie it to `dt`.

## Collision-aware (third-person)
When a wall gets between the camera and the character:
- raycast from the target toward the ideal position.
- if it hits, shorten the distance to the impact point minus a margin.
- smooth the shortening to avoid snaps.
- optional: cross-fade the character to translucent if the camera gets very close.

## Shake and feedback
- apply shake additively on the final transform, not on the target.
- use short duration and exponential decay.
- distinguish damage shake, impact shake, and explosion shake; do not reuse the same profile.
- on mobile, reduce amplitude to avoid nausea.

## FOV
- FOV is a game parameter, not a lost constant.
- Dynamic FOV is useful for speed feedback (sprint, boost). Use carefully, with soft limits.
- In portrait vs landscape on mobile, reconsider FOV and framing (see also `ui-hud.md` for safe areas).

## Aspect and resize
- `aspect = width / height` and `updateProjectionMatrix()` on every resize.
- For orthographic cameras, also update `left/right/top/bottom`.
- Avoid aspect changes per frame; centralize them in a single `resize` path (see `architecture.md`).

## Multiple cameras
- always have one main gameplay camera.
- secondary cameras for minimap, portals, reflections: see `render-targets.md` and `render-target-families.md`.
- debug camera (free-fly) hidden behind a flag. Useful for inspecting the scene without touching gameplay.

## Camera input
- abstract it into a controller with logical axes (`aimX`, `aimY`, `zoom`).
- map keyboard/mouse/gamepad/touch to those axes afterward (see `input-controls.md`).
- user-configurable and persisted sensitivity and invert settings (see `persistence-save.md`).

## Follow-behind + optional offset (free-look without coupling gameplay)

Useful when **another mechanic** (balance, secondary aim, push direction, etc.) must use a stable reference frame, but you still want the camera **not to be fixed**.

Pattern:
1. **Base follow yaw** anchored to the character orientation (for example, `π − facing` to stay behind in the usual +Z/XZ convention).
2. Optional **yaw/pitch offsets** that only exist while the player holds a *look-around* button (or while dragging).
3. On release, **exponential decay** of the offsets toward 0 (λ ~5 s⁻¹: return in ~200–400 ms). The camera returns behind the character by itself without an explicit key step.

What this gains:
- Movement and other mechanics that use a fixed frame **do not depend on camera yaw**; the player does not “break” controls by looking around.
- Pointer lock is not required; the cursor can stay visible (see `input-controls.md`, hold-to-look).

What to watch:
- If movement remains camera-relative, the offsets also rotate the meaning of “forward”. To avoid this, either movement is **world- or character-relative**, or the camera only **orbits visually** while gameplay uses the character's `facing`.
- Update order: calculate the character's **facing / velocity before** positioning the camera if follow yaw depends on `facing`, to avoid introducing an avoidable frame of lag.

## Pause, cutscenes, and takeover
- clear states: gameplay, cinematic, menu, photo.
- in cinematic, gameplay input is muted; camera consumes the timeline.
- transitions between states use a short blend, not a hard cut, unless that is the intended effect.

## Performance
- One additional active camera is another render pass if RTT is used. Evaluate the cost.
- Poorly adjusted orthographic shadows for a top-down camera are the first hitch in this genre.
- Do not use `frustum.containsPoint` as a gameplay tool; it is for culling.

## Debug
- helpers: `CameraHelper`, target visualization, collision raycast line.
- overlay with FOV, position, distance to target, state (gameplay/cinematic).
- free-fly toggle for inspection.

## Anti-patterns
- camera as a child of the character mesh
- direct `camera.lookAt(player.position)` every frame without smoothing
- shake applied to the target instead of to the final transform
- hardcoded constant FOV in three different places
- `lerp` with a fixed factor and no `dt`
- not limiting pitch in first-person (the view flips upward)
- collision-aware logic that teleports the camera when detecting a wall
- same camera for gameplay and minimap sharing a transform

## Strong recommendation
Model from the start:
- `CameraRig` with camera state (`gameplay`, `cinematic`, `debug`).
- separate target and final transform.
- damping by dt and external parameters.
- optional collision-aware raycast.
- input mapped to logical axes.

## Related references
- `architecture.md`
- `character-locomotion.md`
- `input-controls.md`
- `render-targets.md`
- `lights-shadows.md`
