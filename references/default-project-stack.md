# Default Project Stack

## Goal
Provide a sufficiently opinionated default stack for starting Three.js games without wasting time on the same baseline decisions every time.

## Main rule
**Default does not mean mandatory.**
It means “this is what we would use unless the game asks for something else.”

## Recommended default
### Base
- pure Three.js
- Vite
- TypeScript by default
- plain HTML/CSS for the shell and simple UI
- no React/R3F by default

## Why this default
- pure Three.js keeps control and clarity
- Vite provides a dev server, imports, builds, and healthy asset handling without bringing in a large framework
- TypeScript pays off well once the project starts growing
- simple HTML/CSS avoids introducing a UI framework too early

## When to drop down to plain JavaScript
You can use JS instead of TS when:
- it is a very short prototype
- the user wants maximum speed over rigor
- the team truly does not want TS

But the defensible default is still TS.

## Recommended folder structure
```text
project/
  AGENTS.md
  index.html
  package.json
  public/
    models/
    textures/
    audio/
  src/
    main.ts
    app/
      bootstrap/
      config/
      loop/
    game/
      entities/
      systems/
      gameplay/
      levels/
    render/
      scene/
      cameras/
      lighting/
      materials/
      post/
    physics/
    input/
    ui/
    assets/
    utils/
```

You do not need to create every folder on day 1, but this should be the direction.

## Healthy minimum bootstrap
- renderer, camera, and scene initialization
- resize
- central loop
- minimum asset preload
- first playable state before menus or meta-systems

## Default physics
If the game needs real physics:
- **Rapier** as the recommended default

A good choice when you want:
- solid performance
- a reasonable API
- serious collisions and rigid bodies
- an option that is already fairly established on the web

If the game only needs simple collisions or overlaps:
- do not add Rapier by inertia
- start simpler

## Default multiplayer
Healthy default:
- **singleplayer first**
- do not add networking until the base loop works and the game proves it needs it

If multiplayer is core to the concept:
- design it early, but do not marry the project to a library reflexively
- treat solutions like MavonEngine or similar as **candidates to validate in a spike**, not as automatic dogma yet

If multiplayer truly lands later:
- evaluate a **monorepo** early with `client/` + `server/` + `shared/`
- move only what genuinely must match on both sides into `shared/`
  (gameplay constants, validation, wire shapes)
- do not force a monorepo on day 1 if the game is still strictly local

## Assets and shell
- `public/` for simple static assets
- coordinated asset loaders and registry in `src/assets/`
- do not hide game logic inside UI components

If the game has editable levels or authoring data:
- consider an explicit data folder early (`public/levels/`,
  `src/game/levelDefinition.ts`, etc.)
- do not bury critical playable layout in scattered constants inside rendering code

## Scope defaults
When starting:
- one playable loop
- one test scene or level
- one main camera
- one central mechanic

## Anti-patterns
- raw HTML+JS when the project is already going to grow
- adding React out of habit
- adding backend or multiplayer before the core loop
- adding Rapier when simple collisions were enough
- chaotic folder structure from day one

## Strong recommendation
For most new games:
- pure Three.js
- Vite
- TypeScript
- Rapier only if physics really matters
- singleplayer first unless there is a clear multiplayer requirement

When the project moves into real multiplayer or internal tooling:
- root client + `server/` + `shared/` is a healthy evolution of the base stack
- `AGENTS.md` should reflect that structural jump as soon as it happens

## Related references
- `game-kickoff-planning.md`
- `project-agents-md.md`
- `default-content-sourcing.md`
