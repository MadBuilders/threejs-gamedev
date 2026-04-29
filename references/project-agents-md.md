# Project AGENTS.md

## Goal
Use an `AGENTS.md` inside each game as operational memory for the project, recording decisions, changes, conventions, and context that should not live only in chat.

## Main rule
**If a decision matters tomorrow, write it today.**
Do not trust magical memory for why a library was chosen, which folder is authoritative, or which bug is still pending.

Sibling rule:
**if the project changed phase, refresh `AGENTS.md`.**
A useful `AGENTS.md` is not a frozen kickoff snapshot forever. If the game
now has multiplayer, an internal editor, audio, a new folder layout, or a
different phase, the operational memory must reflect it.

## When to create it
Create `AGENTS.md` almost from the beginning once the game moves from a loose idea to a real project.

## What it should contain
### 1. Game context
- provisional name
- premise in one sentence
- target platform
- singleplayer or multiplayer

### 2. Chosen stack
- pure Three.js
- Vite/TS or JS
- Rapier yes/no
- other important libraries

### 3. Project conventions
- folder structure
- naming
- how builds or tests are run, if they exist
- how assets are organized

### 4. Important decisions
Examples:
- why no UI framework is used
- why multiplayer was left out of v0
- which camera system is authoritative
- what physics includes and what it does not

### 4.5 Current phase
Very useful to make explicit:
- current project phase
- what is in scope now
- what is intentionally out of scope

This helps enormously with not trying to do everything at once.

### 5. Short change log
It does not need to be a kilometer-long diary.
It does need to leave a trace of:
- important changes
- new systems added
- serious refactors
- known issues
- clear next steps

### 6. Satellite documents when the project grows
When a subsystem stops being “one more note” and starts having its own roadmap
(for example multiplayer, economy, internal tools, or content pipeline), create
a sibling document and link it from `AGENTS.md`.

Healthy pattern:
- `AGENTS.md` keeps the project-wide picture
- `MULTIPLAYER.md` stores networking decisions, authority, smoke tests, and hosting
- other similar documents if another subsystem becomes just as dense

## Recommended format
```markdown
# AGENTS.md

## Game
- name:
- premise:
- target:
- mode:

## Stack
- three.js:
- vite:
- typescript:
- physics:
- multiplayer:

## Conventions
- structure:
- assets:
- render loop:

## Active decisions
- ...

## Recent changes
- 2026-04-17: initial bootstrap and base loop created
- 2026-04-18: character controller added

## Next steps
- ...
```

## What not to include
- endless logs for every tiny thing
- secrets or credentials
- copies of public documentation
- vague opinions without operational impact
- obsolete project snapshots that no longer describe the real code

## Relationship to memory and the skill
This `AGENTS.md` does not replace the skill.
It grounds the skill in a concrete game.

## Strong recommendation
When a new game starts:
- create `AGENTS.md`
- write the initial stack and decisions
- write the current phase and explicit exclusions
- keep recording relevant changes and next steps

And when the project matures:
- review whether the current phase is still true
- remove contradictions with the real code
- link specialized documents if they already exist

That saves an enormous amount of lost context between sessions.

## Related references
- `game-kickoff-planning.md`
- `default-project-stack.md`
- `default-content-sourcing.md`
