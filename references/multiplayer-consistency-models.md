# Multiplayer Consistency Models: Rollback, Lockstep, Hit Validation

## Goal
Choose and apply the right consistency model for a multiplayer game with Three.js on the client, without mixing visual presentation with simulation authority.

## Main rule
**Not every genre needs rollback.**
And adding rollback where it does not belong can make the project far more complex than the benefit is worth.

Three.js is still the presentation layer here. The consistency model lives in networking and simulation.

## Three main families
### 1. Authoritative server with snapshots and interpolation
This is the healthy default for many games.

Works well for:
- general action
- cooperative PvE
- shooters that are not hyper-demanding
- games with moderate shared physics

Pattern:
- the server decides the real state
- the local client may predict only what is needed
- other clients interpolate snapshots

### 2. Rollback netcode
Best fit when:
- the game depends heavily on precise, fair inputs
- there are few relevant actors per frame
- the simulation can be re-run deterministically or with enough stability

Very common in:
- fighting games
- 1v1 or small 2v2 versus games
- very input-sensitive action

Cost:
- store input and/or state history
- re-simulate
- support frequent corrections
- separate logic, FX, and audio very cleanly to avoid duplicating visual chaos

### 3. Lockstep
Fits when:
- the simulation can advance through synchronized commands
- the pace can tolerate waiting for remote inputs
- determinism is a serious requirement

Very common in:
- strategy games
- tactical games
- low-frequency games or hybrid turn-based games

Cost:
- strict determinism discipline
- poor fit for fast action in the browser unless carefully designed
- sensitivity to desyncs

## Default by genre
### Very precise fighting / duel games
- consider rollback first
- keep the presentation layer strongly decoupled from simulation
- treat FX and animations as re-applicable consequences, not authority

### 3D shooter / competitive action
- authoritative server
- snapshots + interpolation
- limited local prediction
- authoritative hit validation
- partial rollback or server lag compensation, not total client rollback as dogma

### RTS / tactics
- lockstep or command-based variants if the simulation supports it
- otherwise, authoritative server with more abstract snapshots

### Coop / sandbox
- snapshots + interpolation + moderate local prediction
- avoid full rollback unless there is an extremely strong reason

## Client rollback: healthy rules
If rollback is used:
- separate `simulationState` from `presentationState`
- keep history by tick
- re-simulate only the necessary part
- re-trigger visual FX carefully so flashes, particles, or sounds are not duplicated
- isolate random and time so determinism does not break

Healthy pattern:
1. local input enters with a tick
2. provisional simulation runs
3. remote confirmation or correction arrives
4. base state is restored
5. pending ticks are re-simulated
6. presentation is smoothed if needed

Toxic pattern:
- using rollback without a clear fixed tick
- mixing visual state with reversible logical state
- firing audio/particles without control and replaying them on every re-simulation

## Lockstep: healthy rules
If lockstep is used:
- discrete inputs or commands per tick
- identical initial state
- the same deterministic logic for everyone
- random controlled by seed/tick
- avoid depending on browser timing or chaotic floats without control

Three.js must not decide anything critical here.
It only represents the result of the agreed tick.

## Hit validation
### Main rule
**The client may propose a hit. The server decides whether it counts.**

Especially in shooters or competitive action:
- the client may send a firing intent
- maybe origin, direction, tick, expected target
- the server validates using its authoritative state or controlled lag compensation

## Lag compensation
Useful when the genre rewards aim and reaction time.

Typical pattern:
- the server keeps a short history of authoritative positions
- when validating a shot, it reconstructs the approximate state at the relevant moment
- it decides the hit within that window, not only against the server's “now”

Risks:
- overly generous windows
- the feeling of dying behind cover
- inconsistencies if history is poor or the tick is not well defined

To turn this into different policies by weapon or family, see `server-rewind-weapons.md`.

## What to send for hit validation
Prefer sending:
- `tick`
- `shooterId`
- origin or muzzle if applicable
- direction or ray
- weapon/action type
- minimum required context

Do not send as truth:
- “I hit them, subtract 40 HP”
- visual ragdoll state
- final result already precomputed by the client

## Sensible minimum anti-cheat
There is no need to promise the impossible, but do avoid naive designs.

At minimum:
- authoritative server for health, damage, cooldowns, important positions, or derived validation
- plausible movement/input limits
- validation for weapon and action cadence
- rejection of impossible messages or messages outside a reasonable tick range

For an additional layer of telemetry, suspicion scoring, and gradual mitigations, see `anti-cheat-anomalies.md`.

## Presentation firewall
Very important with Three.js:
- local visual impacts can be shown immediately
- real damage, death, or important confirmation should wait for authority
- do not mix a nice hitmarker with gameplay truth without an intermediate layer

## Useful structure
- `inputBuffer`
- `snapshotBuffer`
- `predictionSystem`
- `reconciliationSystem`
- `hitValidationProtocol`
- `lagCompensationStore` if the genre needs it

## Common mistakes
- assuming rollback is always “more pro”
- trying lockstep with a non-deterministic simulation and then praying
- giving the client total authority over damage or hits
- mixing Three.js FX with reversible simulation state
- not separating shot validation from local shot presentation

## Strong recommendation
Choose the model based on genre and maintenance cost:
- general 3D action: snapshots + limited prediction + authoritative hit validation
- fighting games or critical input: rollback
- strategy/tactics: lockstep if the simulation supports it

## To expand later
- rollback with complex physics
- reconciliation of persistent projectiles
