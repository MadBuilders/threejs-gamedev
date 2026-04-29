# Multiplayer

## Goal
Use Three.js as the presentation layer inside a multiplayer game, without turning the scene graph into the source of truth for network state.

## Main rule
**Three.js does not solve multiplayer.**
It solves rendering, cameras, materials, geometry, animation, and the scene graph. Networking, authority, reconciliation, snapshots, and consistency belong in another layer.

## Recommended default
For most games with free movement, action, or collisions:
- **client-server**
- **authoritative server** for important state
- client with smooth local representation
- interpolation for remote entities
- prediction only where it is truly worth it

To decide when to move from this default to rollback, lockstep, or more formal hit validation by genre, see `multiplayer-consistency-models.md`.

If the project is small or turn-based, it can be simplified. But do not start by treating every `Object3D` as if it were a full network entity.

## Mandatory separation
Separate at minimum:
1. **network state**
2. **simulation/game state**
3. **presentation state**
4. **scene graph**

Healthy pattern:
- the network brings messages or snapshots
- gameplay translates them into game state
- visual systems update `Object3D`, animation, particles, and camera

Toxic pattern:
- receive packet
- call `mesh.position.copy(...)` directly all over the app
- use the scene graph as the basis of gameplay

## Scene graph is not authority
Do not use `Object3D.position`, `quaternion`, or hierarchies as the main truth of the shared world.

Better:
- keep entities with stable ids
- separate serializable state
- scene nodes as a derived view

Healthy example of a network entity:
- `id`
- `type`
- `position`
- `rotation`
- `velocity`
- `animationState`
- `health`
- relevant gameplay flags

Do not send:
- references to meshes
- materials
- arbitrary scene graph nodes
- giant objects without a clear schema

## Authority
Choose early what decides the real outcome.

### Authoritative server
Useful for:
- shooters
- real-time action
- shared physics
- competitive games

Advantages:
- fewer easy cheats
- centralized collisions and damage
- consistent rules

Cost:
- more complexity
- reconciliation
- visible latency if it is not smoothed well

Important practical rule:
- score / progress / delivery actions should be validated against
  **state the server already knows**, not against “trusted” payloads
  sent by the client at claim time

Classic example:
- good: the client sends “I want to deliver now” and the server checks the
  distance to the target using its latest known `position` for that player
- bad: the client sends `{ x, z }` and the server validates against those
  freshly received coordinates, because that opens the trivial exploit of
  “I mark the goal without having moved”

### Client-authoritative or peer-ish
I would only recommend it for:
- prototypes
- casual co-op
- slow or low-stakes games
- internal tools

If the game truly matters, an authoritative server is usually the sensible bet.

## Ticks, frames, and snapshots
Do not mix these thoughtlessly:
- browser **render frames**
- **simulation ticks**
- **network updates**

Three.js renders per frame.
The network usually arrives at another rate.
The simulation may run fixed or semi-fixed.

Reasonable default:
- decoupled rendering
- simulation with a defined tick
- remote entities with a short snapshot buffer and interpolation

## Interpolation
For remote entities:
- store recent snapshots
- render slightly in the past
- interpolate between two valid snapshots

That usually looks much better than applying every packet as soon as it arrives.

More concrete pattern:
- short snapshot buffer per entity
- network timestamp or authoritative tick
- presentation time delayed slightly relative to “now”

This lets the client interpolate stably instead of fighting arrival jitter.

## Snapshots
Think of snapshots as compact, serializable state of the world relevant to that client.

Depending on the genre, it may be useful to use:
- small and simple global snapshot
- snapshot per interest area
- partial entities with optional fields

Rule:
- do not send internal Three.js visual state
- send playable state and derive presentation locally

This also applies to gameplay events:
- if the client needs to send a claim, it should be an **intent** or a
  small request
- do not use the claim as a vehicle to smuggle in new authority the server has
  not previously observed

Typical good fields:
- `tick`
- `entityId`
- `position`
- `rotation`
- `velocity` if it helps interpolate or extrapolate
- relevant discrete states

## Prediction and reconciliation
Only add this when feel requires it.

Useful for:
- local player movement
- very frequent inputs
- actions where latency is too noticeable

Rule:
- predict locally only what is essential
- reconcile with authoritative state without wrecking presentation
- do not extend prediction to the whole game just because

Healthy pattern for a local player:
- store inputs with local tick
- simulate movement or immediately responsive actions locally
- when an authoritative correction arrives, re-simulate from the last confirmed state if needed
- smooth the visual layer so reconciliation does not snap unpleasantly

Toxic pattern:
- correcting with visible teleports all the time
- predicting remote entities too without a clear need
- mixing corrected visual position with logical authority without layers

## Remote entities
Each remote entity should have:
- serializable network state
- local visual representation
- clear spawn, update, and despawn lifecycle

Useful pattern:
- `networkEntityMap: id -> entity`
- visual spawn system by type
- visual update system decoupled from network transport

In somewhat more serious games, also add:
- snapshot buffer per remote entity
- interpolator by entity type
- short extrapolation or freeze policy if data is missing

## Multiplayer animation
Do not replicate mixers, actions, or internal visual details as network truth.

Better to replicate:
- high-level state: `idle`, `run`, `jump`, `attack`, `dead`
- relevant direction or velocity
- discrete events: `fired`, `hit`, `respawned`

For somewhat complex characters, think of this as the output of an animation state machine, not an improvised list of clips.

Then the client resolves:
- concrete clips
- blending
- additive layers
- local visual effects

## Shared physics
If physics matters for gameplay:
- decide where authoritative physics runs
- do not trust that two clients will simulate exactly the same by magic
- use Three.js to visualize, not to decide the real outcome

Three.js can coexist with physics, but it does not replace it.

## Camera and local UX
The camera is almost always local.
It usually does not need replication except in specific cases.

Useful rule:
- replicate player intent and state
- do not replicate every camera or presentation detail

Game feel usually depends much more on good local prediction + a smooth local camera than on sending more camera data over the network.

## Interest management
Not every client needs the entire world all the time.

Interest can depend on:
- distance
- room or zone
- approximate line of sight
- team/faction
- tactical relevance

Rule:
- the network should filter before Three.js has to represent irrelevant junk

This hits especially hard when the world grows, there are many actors, or there are RTT/minimaps that may tempt you to include too much live content.

## Rollback, lockstep, and hit validation
Do not use these words as if they were automatic upgrades.

Healthy rule:
- authoritative snapshots + interpolation remain the general default
- rollback mainly fits games very sensitive to input
- lockstep fits better in strategy or command-based simulation
- authoritative hit validation matters a lot in competitive action even if you do not use full rollback

If the project is already in that zone, read `multiplayer-consistency-models.md` before designing the final network stack.

Quick defaults by genre:
- shooter or competitive action: authoritative server, frequent snapshots, limited local prediction, and interest management
- cooperative PvE: snapshots + interpolation + moderate prediction
- large sandbox: partial snapshot by area and strong interest management
- turn-based or low-frequency: simplify and prioritize state clarity

## Representation policy
Do not spend the same network/presentation budget on everything.

Choose per entity:
- remote players: careful interpolation
- fast projectiles: events + light simulation or clear authority
- secondary props: cheap smoothing or discrete updates

Also separate:
- an entity that does not arrive over the network
- an entity that exists but is not rendered
- an entity that exists and is rendered in simplified form

And keep payloads small and stable:
- ids, compact numbers, enums, and concrete events
- no visual blobs or scene graph dumps

## Common mistakes
- using meshes as the data model
- coupling websocket and scene updates directly
- assuming arrival order will always be clean
- mixing local input with remote state without clear layers
- replicating too much visual information instead of playable state
- trying to solve cheating only on the client
- blocking match lifecycle while waiting for external persistence (DB, API, leaderboard)

## Strong recommendation
For any serious multiplayer game, explicitly create:
- `networkClient`
- `networkStateStore`
- `entityReplicationSystem`
- `presentationSyncSystem`

Three.js should mostly enter in the last layer.

## Concrete recommended stack: Colyseus (TypeScript)

For casual / cooperative / lightweight competitive games (not high-level shooters), [Colyseus](https://colyseus.io/) is a sensible choice as the networking layer. It covers transport (WebSocket), synchronized schema, rooms, player lifecycle, and broadcast with very little code. It is already proven in production for pure Three.js.

**Why consider it as the default:**
- Authoritative server from day one without having to write the protocol by hand.
- Declarative schema (`@colyseus/schema`) that serializes efficiently and hydrates on the client as a live object.
- A single monorepo (Vite/TS client + Node/TS `server/`) with shareable types if you want them.
- Automatic binary patches on each `setPatchRate` (default 50 ms); full state is not resent.

**When NOT to use it**: high-cadence competitive shooter (rollback, hit validation), game with minimal bandwidth budget (turns over DataChannel/UDP), or when you already have your own infrastructure. For those cases, see `multiplayer-consistency-models.md` and consider a custom protocol.

### Colyseus 0.17: concrete gotchas
Colyseus 0.17 introduced API changes that break examples from earlier versions and that the docs do not always make obvious. These are the ones that cost time:

- **`MapSchema` is not a real `Map`**: no `for...of`, no spread, no `[...map]`. Use `forEach((value, key) => ...)`. `.get(key)` and `.set(key, value)` do work.
- **Schema listeners do NOT live on the schema object**. In 0.17, direct `players.onAdd(...)` / `players.onRemove(...)` disappeared. The correct API is:
  ```ts
  import { getStateCallbacks } from '@colyseus/sdk';
  const $ = getStateCallbacks(room);
  $(room.state).players.onAdd((player, sessionId) => { /* ... */ });
  $(room.state).players.onRemove((player, sessionId) => { /* ... */ });
  ```
  The SDK types are weak here; a structural cast through `unknown` solves it without losing real safety.
- **State may not be hydrated when `joinOrCreate` resolves**. Accessing `room.state.players` right after the `await` can return `undefined`. Safe pattern: register callbacks after `joinOrCreate` and, to replay current state for a late subscriber, use `room.onStateChange.once(() => seedAllPlayers())`.
- **Schema needs `useDefineForClassFields: false` in server-side `tsconfig`** (and sometimes client-side, depending on the bundler). Without this, schema decorators fail silently and fields do not synchronize.
- **Express 5 + `@colyseus/ws-transport`**: install `@types/express` explicitly or the server typecheck fails with `Could not find a declaration file for module 'express'`.

### Integration pattern with pure Three.js
The **mandatory** separation between network state, game state, presentation, and scene graph (above) still applies, but with Colyseus it becomes concrete like this:

- **Non-blocking connection**: `connectMultiplayer()` launches in the background; the first game frame does not wait for the network. While there is no handle, a no-op `OFFLINE_MULTIPLAYER_HANDLE` lets the game run in singleplayer (very useful for dev without a server running).
- **Throttled 20 Hz pose**: rendering runs at 60 Hz, but `sendPose()` rate-limits internally to 20 Hz. The frequency is adjusted in one place.
- **Single `MultiplayerHandle`**: the API seen by the rest of the game is ~6 methods (`status`, `selfName`, `selfSessionId`, `sendPose`, `subscribeRemotePlayers`, `dispose`). This encapsulates all of Colyseus and allows swapping to another transport without touching `main.ts`.
- **Separate remote manager**: a `remotePlayers.ts` module subscribes through the handle, maintains `Map<sessionId, RemoteAvatar>` with a snapshot buffer for interpolation (~100 ms behind), and reuses the source/instance pattern from `animation-systems.md` to clone the character model (skinned mesh + tinted materials + its own animation mixer).
- **Deterministic visual identity**: the server assigns a `colorHue` on join from a fixed palette (e.g. 8 well-separated HSL values), not the client. This guarantees consistency across all clients without negotiation.
- **Persistence decoupled from round lifecycle**: if you save leaderboards or results at the end of a round, the game should not wait for the database before moving to scoreboard / next round. Prefer: snapshot results, immediate transition, background persistence with timeout and warning on failure.

### End-of-round persistence: strong rule
If there is persistent ranking, analytics, or remote save:
- playable lifecycle wins
- persistence must be **bounded** (timeout) and preferably fire-and-forget
  from the match-flow point of view

Healthy tradeoff:
- worst case: the leaderboard takes one more refresh to reflect the newly finished round
- much worse: the whole match freezes waiting for a slow DB

### Multi-client smoke test
Before validating visually with two tabs, it is worth running a headless smoke test with two real `Client`s that observe each other. It catches schema and broadcast regressions in <3 s. It is worth having in any serious multiplayer project even if rendering still lags behind.

## To expand later
- multiplayer with complex physics
- UDP / WebTransport for latency-sensitive games
