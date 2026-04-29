# Phased Game Workflow

## Goal
Avoid the classic mistake of trying to build an entire game at once by forcing a phase sequence where the mechanic is validated first and complexity is added later.

## Main rule
**Do not build art, backend, and multiplayer on top of an unvalidated mechanic.**
First prove the game works. Then make it feel good. Then make it pretty. Then make it big.

## When to use this
Use this reference when the user wants to:
- start a new game
- plan the initial roadmap
- decide the order for tackling systems
- avoid scope explosion

## Recommended sequence
### Phase 1. Core loop and mechanic
Goal:
- validate the basic playable fantasy
- check whether the main loop is actually fun

What to do:
- basic movement
- minimal camera
- enough collision
- one central mechanic
- a simple objective or fail state
- ugly placeholders are fine

What not to do yet:
- final assets
- complex UI
- meta progression
- backend
- full multiplayer

Exit criterion:
- the user can play for 1 or 2 minutes and say whether the idea works or not

### Phase 2. Feel and structure
Goal:
- make the game respond well
- clean up the minimum architecture so you are not continuing on mud

What to do:
- improve controls
- improve camera
- tune timing, feedback, and initial difficulty
- organize bootstrap, systems, and folders
- remove bugs that break the base experience

What not to do yet:
- major audiovisual polish
- heavy content expansion

Exit criterion:
- the base slice does not only work; it also starts feeling good

### Phase 3. Presentation and content
Goal:
- replace placeholders where it truly helps
- reinforce visual and sound identity

What to do:
- better 3D assets
- textures and materials
- music and SFX
- clearer UI
- more content variety if the mechanic already holds up

Rule:
- improve first what most changes the perception of the loop, not secondary cosmetics

Exit criterion:
- the game communicates its intent better and no longer feels like only a gray prototype

### Phase 4. Advanced systems
Goal:
- add complexity only after validating the base game

May include:
- persistent progress
- economy/meta loops
- more serious generation
- internal tools
- more complete settings

### Phase 5. Multiplayer, if applicable
Important rule:
- if multiplayer is accessory, it clearly comes after the validated core loop
- if multiplayer is central to the game fantasy, design it early, but implement it as a minimal slice, not as a giant system from day 1

Healthy multiplayer slice:
- minimal connection
- one scenario
- one relevant interaction
- validate latency, authority, and feel before scaling

## How to validate with the user
At the end of each phase, run a short validation:
- what works
- what does not work
- what hurts most right now
- whether it is worth moving to the next phase or iterating on the current one

## Anti-patterns
- doing mechanics, art, backend, and multiplayer at the same time
- polishing assets before checking whether the loop hooks people
- adding networking too early to “get ahead”
- adding more content when the real problem is feel
- not explicitly declaring what is outside the current phase

## Strong operating rule
In every new project:
- declare the current phase
- declare what is outside that phase
- do not jump phases without minimal validation

This fits very well with the project `AGENTS.md`.

## Strong recommendation
If the user says “I want to make an X game,” answer with:
1. short kickoff
2. default stack
3. current phase = core loop
4. first playable slice
5. explicit list of things we are not touching yet

## Related references
- `game-kickoff-planning.md`
- `default-project-stack.md`
- `project-agents-md.md`
