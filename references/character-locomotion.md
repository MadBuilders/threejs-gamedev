# Character Locomotion

## Objective
Design character movement in Three.js with playable feel, maintainable architecture, and clear boundaries between input, locomotion, collision, camera, and animation.

## Main rule
**Locomotion is not just input and not just physics.**
It is its own layer that translates player intent into believable movement, spatial constraints, and character state.

## Recommended separation
Separate at least:
1. **input intent**
2. **character locomotion state**
3. **collision/physics queries**
4. **camera behavior**
5. **animation state**

Healthy pattern:
- input produces intent (`moveX`, `moveY`, `jumpPressed`, `sprintHeld`)
- locomotion decides acceleration, velocity, turning, and ground contact
- collision resolves penetrations and constraints
- animation consumes high-level state
- camera follows or responds without becoming character logic

## Common types
### First-person
- camera anchored to the character
- movement relative to camera yaw
- pointer lock is common on desktop

### Third-person
- camera partially decoupled
- movement relative to the camera projected onto the plane
- usually requires better turning, facing, and animation blending

### Tank / vehicle-lite (no strafe)

When you **do not** want A/D as lateral strafe (for example, the right hand or another mechanic already consumes “lateral” axes, or you want turning the body to be a costly decision):

- **W/S**: push in the character's **forward/back** direction (in its frame), not the camera's.
- **A/D**: change **only yaw** (`facing`) at a constant rate.
- **Facing as state**: update it from turn input, not from `atan2(velocity)` (if facing were derived from velocity, pivots in place or intent “drift” become weird).

Typical parameters to expose: turn speed (rad/s), eventually forward/backward asymmetry if the game's fantasy calls for it (not required).

Advantages: stable scheme with free-look or auto-follow camera; every turn is explicit (useful if another simulation coupled to the character reacts to **angular velocity**).

Tradeoff: there is no strafe; curves are “W + A” or “W + D”, not “D only”.

### Runner / lane / arcade
- more scripted locomotion
- less freedom, more control over feel

Do not mix these types' needs without deciding which one leads.

## Recommended default
For many 3D web games:
- kinematic player
- simple collider, usually a capsule
- static world queried with a spatial structure if needed
- gravity and jump controlled by custom logic
- movement relative to camera
- camera and locomotion decoupled, but coordinated

## Character collider
The `games_fps` example leaves a very useful signal:
- the **capsule** is usually a much healthier base than a box for basic human movement

Advantages:
- climbs small irregularities better
- catches less on corners
- represents a standing body fairly well

Practical rule:
- simple collider first
- complex player collider almost never as the default

## Ground, slopes, and walls
Detecting collision is not enough. You have to classify it.

Useful pattern:
- use the contact normal
- decide whether something counts as ground using a threshold
- handle walls separately

`games_fps` leaves a canonical idea:
- not every collided surface is ground
- slopes need an explicit criterion

## Ground movement vs air movement
Separate clearly:
- ground acceleration
- ground friction or damping
- air control
- gravity

Strong pattern:
- more control and response on the ground
- less control in the air
- jump only from a valid grounded state

The official demo even includes reduced air control. That detail deserves to stay as doctrine because it changes game feel enormously.

## Substeps
When there is speed, gravity, or fast collisions:
- using simulation substeps can prevent many problems

The `games_fps` example does this with clear intent:
- it divides the frame into several steps to reduce tunneling and resolution errors

Rule:
- do not blindly trust a single step per frame once you already see clipping or instability

## Recommended update order
1. read normalized input
2. build locomotion intent
3. apply acceleration and gravity
4. integrate movement
5. resolve collisions
6. update state (`grounded`, `falling`, `jumping`, etc.)
7. synchronize camera
8. emit state for animation

## Locomotion state
Do not stop at `velocity` and call it done.

Useful minimum:
- `grounded`
- `jumpRequested`
- `falling`
- `moveIntent`
- `moveDirectionWorld`
- `speed`
- `facingDirection`
- `sprint`
- `crouch` if it exists

This state should be clean enough to feed animation and multiplayer without exposing raw input details.

## State machine
As soon as the character does more than walk:
- an explicit state machine is useful, or at least well-delimited states

Typical states:
- idle
- locomotion
- jump start
- airborne
- land
- dash
- climb
- knockback

You do not need a mega hierarchy from day one, but you should avoid scattered `if` rules everywhere.

## Root motion vs gameplay-driven movement
Decide early.

### Gameplay-driven locomotion
- actual movement is controlled by the controller
- animation follows along
- usually the healthiest default for web and game prototypes

### Root motion
- animation drives part of the displacement
- useful in specific cases, combat, or authored actions
- requires more discipline for collisions, networking, and blending

Initial recommendation:
- use gameplay-governed locomotion by default
- add root motion only where it provides a lot of value and you know why

## Camera
The camera should not decide character movement by accident.

Useful rule:
- use camera orientation as the reference for intent
- but keep the character's own state for facing and locomotion

In first-person, the connection can be more direct.
In third-person, if you bind everything to the camera without filtering, the character often feels strange or twitchy.

## Pointer lock and lifecycle
The `misc_controls_pointerlock` example reminds us of something important:
- mouse control on desktop has a real lifecycle: lock, unlock, overlay, focus

That is not a minor detail.
Design it as part of the controller, not as a loose patch.

## Physics and queries
Even if you use a physics engine or structures like an octree:
- the player controller deserves specific rules
- do not delegate the entire feel of the character to raw simulation

Healthy pattern:
- custom locomotion
- supporting queries/collisions
- clear synchronization with the visual representation

## Teleport and recovery
Another very real pattern from demos and games:
- if the character falls out of the world, recover it

It seems obvious, but it is worth making explicit:
- have `teleportToSafePoint()` or equivalent
- do not leave state broken after a fall or spatial NaN

## Useful debug
- visualize the player collider
- show contact normal and grounded state
- show horizontal/vertical velocity
- show locomotion state
- toggles for substeps and gravity
- visible respawn or safe points

## Anti-patterns
- joining DOM input, camera, jump, and collisions into one monster function
- using the visual mesh as the player's real collider
- failing to distinguish ground from wall
- depending on fully realistic physics for the main movement
- not separating air and ground
- not having explicit locomotion state
- coupling animation directly to the keyboard instead of to character state

## Strong recommendation
Create a `characterController` or `locomotionSystem` that:
- consumes abstract input
- maintains the character collider and state
- resolves movement and collisions
- publishes locomotion state for camera and animation
- includes recovery, respawn, and debug

## To expand later
- third-person camera rigs
- finer stair stepping
- ledge detection
- selective root motion
- locomotion networking
- combat and advanced locomotion
