# Animation State Machines

## Objective
Turn character animation into an explicit layer of states, transitions, and layers, instead of a collection of clips played by hand.

## Main rule
**Do not drive animation with keys or loose clips.**
Drive it with character states and gameplay intents.

## Healthy separation
Separate at least:
1. **locomotion state**
2. **animation state machine**
3. **clip/actions layer**
4. **additive or partial-body layers**
5. **event-driven one-shots**

Healthy pattern:
- locomotion/controller publishes high-level state
- animation state machine decides the main visual state
- concrete actions activate through controlled transitions
- additive or upper-body layers blend separately

## What a state machine resolves
- which base clip should be active
- when to change state
- which transition to use
- which events trigger one-shots
- which layers can coexist
- which priorities win when there is a conflict

## Typical base states
### Locomotion base
- idle
- walk
- run
- sprint
- jumpStart
- airborne
- land

### Contextual states
- crouch
- aim
- block
- hitReact
- attack
- dead

Not all of them should live in the same machine. Sometimes it is better to have:
- one main locomotion state machine
- one upper layer for combat/interaction

## Recommended default
For a typical playable character:
- one main machine for locomotion
- mutually exclusive base clips
- additive or partial layers for secondary poses
- discrete events for short actions
- centralized transitions with coherent durations

## Base pattern
### 1. Character state
Consume things like:
- `grounded`
- `speed`
- `moveDirection`
- `facingDirection`
- `sprint`
- `jumpRequested`
- `attackRequested`
- `hitReaction`

### 2. Animated state resolution
Example:
- if `!grounded` -> `airborne`
- if `grounded` and `speed` is near zero -> `idle`
- if `grounded` and `speed` is medium -> `walk`
- if `grounded` and `speed` is high + sprint -> `run`

### 3. Clip resolution
- `idle` -> idle clip
- `walk` -> walk clip
- `run` -> run clip
- `airborne` -> jump/fall loop clip

### 4. Layer resolution
- `aim` adjusts upper body
- `hitReact` or a gesture can enter as a one-shot or temporary layer

## Base layer vs additive layer
The official additive blending example leaves very good doctrine:

### Base layer
- locomotion and main body
- only one dominant, or almost dominant, action at a time

### Additive layer
- poses or partial corrections
- continuous weights
- useful for aim, sneak pose, head shake, light reaction

Strong rule:
- do not use additive as a patch to fix a broken base
- first resolve base locomotion properly
- then add layers with a clear purpose

## Full body vs upper body
Very useful game pattern:
- locomotion in the full body, or lower body as dominant
- actions like aim, reload, attack windup, or gesture in upper body

Although Three.js does not ship a ready-made “layer graph” like a full engine, the doctrine still applies:
- conceptually separate full layers from partial layers
- do not let an upper-body action accidentally break base locomotion

## Transitions
Centralize transitions; do not fire them from all over the code.

Good rules:
- short, coherent durations
- reset the incoming clip time when appropriate
- do not chain contradictory fades without control
- if a transition must wait for the end of a loop, synchronize it explicitly

The official example does exactly this with `prepareCrossFade`, `synchronizeCrossFade`, and `executeCrossFade`. Very good signal.

## Instant vs sustained states
### Sustained
- idle
- walk
- run
- airborne
- crouch
- aim mode

### Instant or one-shot
- attack start
- roll
- hit reaction
- short emote
- interact

Rule:
- a one-shot should not destroy locomotion logic if it only needs to overlay or temporarily block it

## Priorities
Define what wins when there is a conflict.

Possible example:
1. dead
2. hard stun / knockback
3. attack locked animation
4. jump/airborne
5. locomotion
6. idle

It does not need to be this exact list, but there should be an explicit policy.

## Root motion
If you use root motion, the state machine has even more responsibility.

Base recommendation:
- gameplay-governed locomotion for normal movement
- selective root motion for special actions if it pays off

Because if everything depends on root motion:
- collisions get more complicated
- multiplayer gets more complicated
- prediction gets more complicated
- tuning feel gets more complicated

## Multiplayer
The state machine helps a lot with not replicating visual junk.

Better to replicate:
- high-level state
- velocity or relevant intent
- discrete events

Do not replicate:
- exact internal weights of every action
- blending details unless there is an extreme need

## Useful debug
- current machine state
- previous state
- transition in progress
- active base clip
- additive layer weights
- lock or priority flags

## Anti-patterns
- `if (keyW) play('walk')`
- transitions fired from input, gameplay, and UI at the same time
- additive layers with no ownership or limits
- not distinguishing one-shots from sustained state
- not defining priorities between attack, jump, hit, and locomotion
- replicating internal mixer details as if they were gameplay

## Strong recommendation
Create a `characterAnimationStateMachine` or equivalent that:
- consumes character state
- resolves the main visual state
- fires centralized transitions
- manages layers and one-shots
- publishes readable debug

## To expand later
- bone masks or upper/lower body strategies
- one-shots with cancel windows
- melee combat
- 8-directional locomotion
- selective root motion by action
