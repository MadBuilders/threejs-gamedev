# Anti-Cheat by Telemetry and Anomalies

## Goal
Add a sensible layer of anomaly detection and plausibility validation, without hand-waving about “perfect anti-cheat” or putting responsibilities on the client that do not belong there.

## Main rule
**First authority and validation, then detection.**
Telemetry does not replace an authoritative server. It complements it.

## What it tries to solve
- detect impossible or highly improbable patterns
- review abuse without blocking legitimate play because of noise
- avoid relying only on simplistic binary rules

## Mandatory minimum base
Before talking about anomalies, have:
- server authority over health, damage, cooldowns, and critical validations
- plausible movement and cadence limits
- rejection of impossible messages

If that does not exist, telemetry arrives too late.

## Useful anomaly types
### Movement
- impossible speed
- repeated improbable acceleration
- teleports outside the rules
- frequent mismatches between expected input and observed result

### Combat
- cadence above the allowed rate
- sustained statistically absurd accuracy
- overly perfect target acquisition patterns
- shots with origin/direction incompatible with posture or weapon

### Economy/actions
- using abilities without resources
- sequences impossible because of cooldowns
- commands outside logical order or a reasonable tick range

## Healthy model
Use layers:
1. **hard validation**: reject the impossible
2. **soft suspicion**: accumulate signals
3. **review or mitigation**: act if the pattern persists

## Suspicion scoring
Better than banning for one isolated event.

Conceptual example:
- each anomaly adds a weight
- there is temporal decay
- certain critical events weigh much more
- final actions require accumulation or very strong evidence

## Possible mitigations
Not everything has to be an immediate ban.

Options:
- ignore the invalid event
- correct position/state
- reduce trust in client reports
- flag the session for review
- apply progressive restrictions

## What to log
Store enough to investigate:
- `playerId`
- anomaly type
- tick/timestamp
- weapon/action context
- summarized metrics
- accumulated suspicion state

Avoid:
- huge scene graph logs
- unnecessary visual data

## False positives
A genuinely delicate topic.

Noise can come from:
- high latency
- jitter
- packet loss
- bugs in the game itself
- client/server frame pacing differences

Rule:
- do not punish harshly from one doubtful signal
- correlate events and context

## What Three.js can do here
Very little for authority, some for observability:
- debug visualization of rays, hitboxes, or trajectories
- internal overlays to investigate odd cases
- visual replay of suspicious events if internal tooling exists

But real detection lives outside rendering.

## Anti-patterns
- promising “anti-cheat” with only client heuristics
- banning for high accuracy without context
- confusing a netcode bug with a real cheat
- logging so much that nobody analyzes it later

## Strong recommendation
Design a small, actionable anomaly layer:
- a few good signals
- simple scoring
- useful logs
- gradual mitigations

## To expand later
- replay-assisted review
- input device anomalies
- correlation between squads/accounts
