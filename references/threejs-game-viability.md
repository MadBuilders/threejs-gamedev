# Three.js Game Viability and Inspiration

## Goal
Help decide what kind of game fits Three.js well, which ideas should be scoped down or prototyped first, and where the areas are that tend to look easy but become expensive or fragile.

## Main rule
**Just because something is possible does not mean it is a good idea to start with.**
Three.js allows a lot, but the useful question at the beginning is not only “can it be done?” but “can it be done with this scope, this team, and this time?”

## How to use this reference
Use it when the user asks:
- whether a game idea is viable in Three.js
- whether something fits well or poorly as a first project
- what kinds of games or prototypes make more sense
- which ideas deserve an early spike before committing to architecture

## Three useful categories
### 1. Good idea to start with
Fits Three.js well and usually allows a quick prototype.

### 2. Viable, but with care
Can be done, but usually hides technical, performance, or content cost.

### 3. Bad idea to start with
Not because it is impossible, but because it tends to explode scope, tooling, or complexity too early.

## Good idea to start with
### Games that usually fit well
- 3D runner or simple arcade
- simple driving or short circuit
- light third-person exploration
- simple shooter with few enemies
- 3D puzzle with one strong mechanic
- small survival game with a very clear loop
- small physics toy box
- stylized/abstract experience with few simultaneous systems

### Why they fit well
- quickly prototypable gameplay loop
- low or moderate dependence on massive content
- relatively controllable camera and controls
- room to use placeholders without breaking the idea

## Viable, but with care
### Large worlds or serious procedural generation
Viable if scoped well, but it forces you to think about:
- streaming
- chunks
- draw call and memory budgets
- content tools

### Competitive multiplayer
Viable, but requires:
- clear authority
- snapshots/prediction/reconciliation
- hit validation
- latency testing

As a first large project, it is usually a bad bet unless the social loop is the heart of the game.

### Relevant continuous physics
Viable, but confirm early:
- whether serious simulation is truly needed
- whether Rapier enters because of real need or reflex
- how much of the gameplay depends on fine physics stability

### Portals, mirrors, refractors, and premium RTT
Viable, but they can eat performance and visual complexity if used as generalized decoration.

### Vehicles, flight, or non-trivial movement
Great for prototyping if the game revolves around it, but validate early:
- camera
- control feel
- collision
- movement stability

## Bad idea to start with
### MMO-ish or huge social world
Not impossible, but a terrible starting point if there is not already a small, proven core loop.

### Hyper-systemic game with crafting, housing, combat, economy, and multiplayer from day 1
That is not a prototype. That is a scope explosion machine.

### Serious competitive shooter with strong anti-cheat, complex backend, and full matchmaking from the start
Viable in theory, but not as the first step of a new project unless the team is very prepared.

### Huge procedural sandbox with complex physics and simultaneous multiplayer
You can dream it, of course. You can also sink the project in the first week.

## Signs that an idea needs an early spike
- it depends on a strange or difficult camera
- it depends on very specific physics
- it depends on premium RTT or portals as the central fantasy
- it depends on latency or synchronization to be fun
- it depends on streaming or large worlds to exist

When one of those signs appears:
- make a small spike first
- do not design the whole game around an unvalidated assumption

## Ideas that usually look good in Three.js
### By aesthetic
- clean low poly
- simple stylized
- geometric abstract
- dreamlike or surreal environments with few assets but good light/color

### By mechanic
- pleasant movement and clear camera
- light physical interaction
- spatial puzzles
- arcade driving
- short, expressive traversal
- action toy with few enemies but good feel

Three.js shines when:
- spatial readability matters
- movement feels good
- the visual direction does not depend on massive photoreal content

## Ideas that require extra discipline
- atmospheric horror, because the feel depends heavily on lighting/audio/pacing
- builders or sandboxes, because they demand tooling and scalable content
- shooters, because weapon feel and hit validation matter a lot
- simulations, because they look like “they already work” long before they are actually good

## Healthy inspiration rule
When taking inspiration from examples or demos:
- take the technical pattern, not a promise of automatic production readiness
- extract a feeling or mechanic, not the whole implicit scope
- translate the idea into a small playable slice

## Useful questions to ground an idea
- what is the main loop in one sentence?
- can it be proven fun with placeholders?
- what is the riskiest technical system to validate first?
- what can I leave out for two weeks without breaking the central fantasy?

## Strong recommendation
When an idea arrives green:
1. classify it as good to start with, viable with care, or bad to start with
2. detect the central technical risk
3. define a minimum slice that tests only that fantasy
4. explicitly state what is parked outside the first phase

## Anti-patterns
- answering “yes, anything is possible” without talking about scope
- selling multiplayer or massive procedural generation as the obvious next step
- confusing a flashy example with a cheap feature
- trying to validate the idea with final assets instead of with mechanics

## Related references
- `game-kickoff-planning.md`
- `phased-game-workflow.md`
- `default-project-stack.md`
