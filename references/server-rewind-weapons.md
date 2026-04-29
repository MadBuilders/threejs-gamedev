# Server Rewind by Weapon

## Goal
Apply lag compensation or server rewind deliberately per weapon and per action, instead of treating every shot as if it needed exactly the same temporal validation.

## Main rule
**Not every weapon deserves the same rewind window.**
A fast hitscan weapon, shotgun, slow projectile, or melee attack does not ask for the same reconstruction of the past.

## What it tries to solve
- validate hits more fairly
- avoid absurd windows that hand out free shots
- reduce the feeling of dying behind cover
- avoid using a single rewind policy for the whole arsenal

## Base model
Typical pattern:
- the client sends a fire attempt with `tick`, origin, direction, and weapon
- the server queries recent authoritative history
- it reconstructs the approximate state at the relevant moment
- it validates according to that weapon's policy

## Separate by weapon family
### 1. Precise hitscan
Examples:
- rifle
- accurate pistol
- sniper

Usually deserves:
- short but reliable rewind
- strict line/ray validation
- strong maximum-window limits

Risk:
- if the window is too generous, it feels unfair to the player who was already behind cover

### 2. Shotgun or multi-pellet
Usually requires:
- the same rewind base as hitscan
- validation of multiple rays or deterministic/server-side spread
- more care with cost due to the number of potential hits

Risk:
- replaying the entire simulation pellet by pellet in an expensive and poorly controlled way

### 3. Slow projectiles
Examples:
- rockets
- slow arrows
- energy balls

Often they do not need strong rewind for the final impact.
What usually matters is:
- validating the initial spawn
- plausible initial velocity
- authoritative projectile simulation or strong correction

### 4. Melee
Usually needs something else:
- short window
- spatial validation by volume or arc
- temporal confirmation consistent with the animation or attack tick

### 5. AoE or explosives
Separate:
- validation of the impact/explosion point
- application of radial damage

Not everything is raycast rewind.

## What to store in history
Store only what is necessary, with a small window.

Usually:
- authoritative transform by tick or timestamp
- relevant posture if it affects the hitbox
- alive/dead/active state
- maybe simplified hit volumes

Avoid:
- reconstructing the whole scene graph
- depending on the client's visual state

## Maximum window
Have a clear maximum window per weapon or family.

Conceptual example:
- precise rifle: stricter
- shotgun: strict but tolerant of spread
- melee: very short
- slow projectile: minimal or focused on spawn

## Fairness vs feel
Real tradeoff:
- a larger window helps high-latency players
- a larger window also increases deaths that feel “unfair” to the victim

Rule:
- tune by genre, TTK, and game pace
- do not copy a universal window from another game

## Minimum shot data
Send:
- `weaponId`
- `tick`
- origin or output socket if applicable
- direction or intent
- maybe a seed if spread needs reproducibility

Do not send as truth:
- final list of valid hits
- definitive damage
- resolution already precomputed by the client

## Cost and budget
Rewind has CPU and memory cost.

Measure:
- number of validations per second
- cost per weapon
- size of live history
- impact on spikes under intense combat

## Healthy defaults
- short, explicit rewind
- policies per weapon, not a single global policy
- server decides final damage
- hitscan and melee stricter than diffuse or slow weapons

## Anti-patterns
- a single rewind window for the entire arsenal
- using huge rewind to hide weak netcode
- validating damage directly from the client
- reconstructing too much state per shot

## Strong recommendation
Create a policy table by weapon or family:
- `maxRewindMs`
- `validationMode`
- `spreadPolicy`
- `allowPastCoverGrace` if applicable

## To expand later
- variable hitboxes by posture
- hybrid persistent projectiles
- server rewind with environment destruction
