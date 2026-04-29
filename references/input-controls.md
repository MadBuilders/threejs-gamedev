# Input and Controls

## Objective
Design a robust input layer for pure Three.js games without coupling gameplay directly to browser events.

## Main rule
Create an **input abstraction layer**.

Gameplay should not depend directly on `keydown`, `pointermove`, `touchstart`, or `gamepad`. It should depend on more stable input actions or states.

## Recommended separation

1. **raw capture**
   - keyboard
   - mouse
   - touch
   - gamepad
   - sensors if needed later

2. **normalization**
   - convert events into common states or actions
   - example: `moveLeft`, `jump`, `interact`, `tiltX`

3. **consumption by systems**
   - player controller
   - camera controller
   - UI controller
   - debug tools

## Useful principle
Think of input as an internal game API, not as a collection of loose listeners.

## Recommended patterns

### Continuous states
For movement and camera control, prefer continuous states:
- `moveX`
- `moveY`
- `lookX`
- `lookY`
- `tilt`

### Discrete actions
For one-off events:
- `jumpPressed`
- `interactPressed`
- `pausePressed`

### Configurable mapping
Leave room to remap sources:
- keyboard on desktop
- touch on mobile
- gamepad if applicable

## Touch and mobile
Do not assume mobile controls are a literal translation of keyboard controls.

Design with:
- clear touch zones
- visual feedback
- tolerance for large fingers
- less fine precision than with a mouse
- avoiding any dependency on hover

The manual also leaves a practical detail that belongs here: if the canvas needs keyboard input, think explicitly about focus and input capture. Do not assume the canvas receives keyboard input just because it is on screen.

### Two virtual joysticks (reusable pattern)

For gameplay with two continuous axes (movement + something else: aiming, balancing, turret rotation, object balance), the classic mobile scheme is **two dynamic-center joysticks** in the lower corners. Concrete gotchas that save time:

- **`setPointerCapture(e.pointerId)` per zone, not on the canvas**. If the listener lives on the canvas or you do not capture, iOS Safari routes the second finger's `pointermove` to whichever element got the most recent capture → diagonal *movement + second action* breaks. Each zone stores its own `pointerId` and calls `setPointerCapture` on itself. This is the real requirement for stable multi-touch.
- **Dynamic center > fixed corner**. The stick base appears where the finger lands inside the zone. It tolerates thumbs starting from any position and does not force the player to aim at a specific anchor.
- **Dead zone and clamp defaults**: maximum radius ~60 px, dead zone ~0.12 per axis (without it, a resting thumb injects a trickle of continuous input). Normalize to `[-1, 1]` before handing values to gameplay.
- **Touch as an override on the same `axes` struct**, not a parallel pipeline. `update()` reads keyboard by default and, only if the stick is `active`, overwrites the corresponding axis:

```text
// pseudo update()
axes.moveF = kbd(w) - kbd(s);                  // default
if (touch.left.active) axes.moveF = touch.left.y;   // override
```

Gameplay consumes a single struct, and the desktop code stays intact without scattered `if (isTouch)` checks.

- **Hold-to-look and two sticks are mutually exclusive on mobile.** Do not also "paint" the canvas `pointerdown` on top of the sticks: remove it when input runs in touch mode, or a third finger outside both zones triggers free-look with no clean way to release it. General rule: any global pointer capture disappears in touch.
- **`touch-action: none` + `user-select: none` on the `canvas` (not on `body`)** so iOS Safari does not interpret dragging as text selection, pinch-zoom, or scroll. On `body`/`html`, `-webkit-tap-highlight-color: transparent` is enough for visual polish.
- **Device class once at boot**: `matchMedia('(pointer: coarse)')` → `body.classList.add('is-touch')`. CSS (zones visible only on coarse pointers) and JS (constructing / skipping the sticks) key off the same class. This avoids `matchMedia` scattered through every module and keeps the decision in one place.

## Raycasting and interaction
If the game needs to select or touch 3D objects:
- centralize `Raycaster`
- separate picking from gameplay
- do not spread raycasts across twenty different systems
- convert raycast results into manageable game events

Healthy pattern:
- convert pointer or touch to normalized coordinates once
- resolve picking in a dedicated system
- emit results that gameplay, UI, or debug systems can interpret

Important detail from real examples:
- if the canvas does not fill the window, normalize the pointer against `renderer.domElement.clientWidth/clientHeight` or against the canvas's real rect, not reflexively against `window.innerWidth/innerHeight`
- if the use case is bounded, raycast against specific targets instead of the whole scene

## Camera and controls
Clearly distinguish:
- debug or editing camera controls
- gameplay camera controls
- player controls

Do not mix prototype `OrbitControls` with the final game camera without marking the difference.

The official examples are useful here, but they leave a clear lesson: many demos use controls to teach a technique, not to represent a final game control scheme. Copying the whole example without separating that intention usually dirties the architecture.

If you use pointer lock on desktop, treat its lifecycle as part of the design:
- lock
- unlock
- focus
- overlay or instructions

Do not assume pointer lock is just one line of code with no UX implications.

### Hold-to-look without pointer lock (visible mouse)

When you want to **look around** with the mouse but:
- the game does not require continuous fine aiming, and
- you prefer **not** to hide the cursor or require permanent click-to-play,

a healthy alternative is to **hold down** a button (usually `pointerdown` on the canvas with `setPointerCapture` so you keep receiving `pointermove` even if the cursor drifts slightly outside).

Practical rules:
- Accumulate `movementX/Y` only while the button is down.
- Listen for `pointerup`/`pointercancel` on `window` and on the canvas, plus `blur`: release the button even if focus is lost.
- If the camera applies an **offset that decays on release**, most players do not need an extra “reset view” action.

This pairs well with **follow + offset decay** cameras (see `cameras.md`).

## Gamepad
Design for gamepad support if the game type benefits from it, but do not force it from day one.

Useful rules:
- read state every frame
- apply dead zones
- normalize axes
- do not assume identical layouts across controllers

## Suggested structure

```text
systems/
  inputSystem.js
  pointerSystem.js
  gamepadSystem.js
controllers/
  playerController.js
  cameraController.js
```

## Anti-patterns
- gameplay connected directly to DOM listeners
- duplicating logic for keyboard and touch instead of normalizing
- putting raycasting inside every interactive entity
- using debug controls as if they were production controls
- failing to distinguish continuous input from one-off actions
- assuming focus, pointer lock, or keyboard input are already solved without designing them

## Control design checklist
- does it work on desktop?
- does it work on mobile?
- does the camera compete with the primary control?
- is input decoupled from the DOM?
- are action names clear?
- is it easy to change the scheme later?

## To expand later
- gyroscope and sensors
- input buffering
- rebinding
- accessibility and alternative schemes
- control patterns for third-person, runner, and balance games
