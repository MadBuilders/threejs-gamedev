# Game Kickoff Planning

## Goal
Provide a more concrete start when the user says “I want to make an X game,” avoiding the jump into code before scope, stack, and the first playable slice are settled.

## Main rule
**Do not start with the folder or the shader.**
Start with a few decisions that change architecture, scope, and tooling.

## When to use this
When the user wants to:
- start a new game
- prototype a game idea
- decide the initial stack or structure
- ground a still-vague concept

## Recommended flow
1. Ask a few questions, but only the ones that actually change the project.
2. Summarize the answers in a short brief.
3. Recommend the default stack and structure.
4. Declare the current phase and what is explicitly out of scope.
5. Choose a very small first playable slice.
6. Create or propose a project `AGENTS.md` to record decisions and changes.

## Questions worth asking
Do not run an endless interrogation. Usually 8 to 12 are enough.

### Game and scope
- what genre or genre mix is it?
- what does the player do in the main loop, in one sentence?
- is it singleplayer or multiplayer?
- what ambition level does it have: short prototype or serious project?

### Camera and controls
- first person, third person, top-down, side view, free camera?
- keyboard/mouse, gamepad, touch, or a mix?
- desktop, mobile, or both?

### World and presentation
- full 3D, 2.5D, or very simple?
- placeholder, low poly, stylized, realistic, abstract?
- are important physics involved, or only simple collisions?

### Production
- what must be playable first?
- what can safely remain placeholder?
- are there asset, license, or time constraints?

## What to return after asking
Answer with a short kickoff brief, not an essay.

### Useful shape
- game concept
- target platform
- camera/control
- singleplayer or multiplayer
- recommended stack
- main risks
- first playable slice
- what is explicitly left out of v0

If the idea is still very green or its scope smells off, compare it against `threejs-game-viability.md` before committing to a stack or roadmap.

## Recommended initial slice
The first slice should be almost ridiculously small.

Examples:
- character moves + camera + basic collision + simple objective
- vehicle controls + minimal track + restart
- simple shooter with one weapon and one dummy
- puzzle with a single main mechanic

## Recommended work order
Do not try to do everything at once.

Healthy pattern:
- phase 1: core loop and mechanic
- phase 2: feel and structure
- phase 3: presentation and content
- phase 4: advanced systems
- phase 5: multiplayer if applicable

For this full sequence, see `phased-game-workflow.md`.

## Anti-patterns
- starting with menus, settings, backend, and persistent progress before the core
- designing twenty systems before the first playable loop
- adding multiplayer on day 1 without validating that the game needs it
- asking about everything when the real genre is not clear yet

## Strong recommendation
When the user says “I want to make an X game,” guide it this way:
- settle a short brief
- recommend the default stack
- define a playable v0
- leave backlog outside the starting scope
- create the project `AGENTS.md`

## Related references
- `phased-game-workflow.md`
- `threejs-game-viability.md`
- `default-project-stack.md`
- `default-content-sourcing.md`
- `project-agents-md.md`
