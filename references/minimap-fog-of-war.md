# Minimap Fog of War

## Goal
Build useful minimaps with fog of war without turning the system into another expensive camera that renders the whole world out of habit.

## Main rule
**The minimap should prioritize readability and tactical state, not visual fidelity.**
Fog of war belongs more to a visibility/gameplay subsystem than to pretty rendering.

## What it usually really needs
- simplified geometry or base map
- player or team position
- explored zones
- zones visible now
- relevant markers

It usually does not need:
- full world materials
- complex shadows
- postprocessing
- complete cosmetic props

## Two useful layers
### 1. Base map layer
Can come from:
- simplified orthographic camera
- prebaked texture
- tactical chunks/proxies

### 2. Visibility layer
Represents:
- currently visible
- previously explored
- unknown

This layer can update through its own logic and does not need to come from rendering the whole world every frame.

To decide how to represent that layer with readable masks and blending, see `fog-mask-blending.md`.

## Healthy model
Think of the minimap as a combination of:
- static or cheap world representation
- dynamic visibility overlay
- icons/markers for relevant entities

## Reasonable implementations
### Option A: minimap RTT + fog overlay
- render target with simplified orthographic view
- additional texture or mask for fog
- simple final composition

### Option B: prebaked map + dynamic fog
Often the healthiest option.

- base map image or texture
- world -> map coordinate system
- dynamic exploration/visibility mask
- separately updated icons

### Option C: tactical chunks
Useful in large worlds:
- base map by chunks
- visibility by sector
- local updates, not global

## What to track
Separate at least:
- `currentlyVisible`
- `explored`
- `neverSeen`

That enables classic fog:
- visible: clear
- explored: dimmed
- unseen: hidden

## Healthy update policy
Fog does not always need 60 fps.

Options:
- update by tactical tick
- update after moving a minimum distance
- update only if a relevant revealer changes
- recalculate partially by zone/chunk

## Visibility sources
Depending on the game:
- radius around the player
- simplified raycasts
- room-based visibility
- grid or nav sectors
- influence from allied units

Do not tie this directly to render target cost. Decide the visibility logic first.

## Chunks and large worlds
In large maps:
- do not keep everything at uniform detail
- split exploration by tiles/chunks/sectors
- serialize exploration state separately from rendering
- load only needed overlays nearby or in active UI

## Integration with RTT
If the minimap uses a camera:
- orthographic by default
- modest resolution
- filtered layers
- update independent from the main view

Fog should survive even if you lower the RTT frequency a lot.

## Integration with gameplay
Fog of war is not just decoration.
It can affect:
- visible markers
- detectable enemies
- known objectives
- tactical navigation

That is why fog state should live outside Three.js, and Three.js should only paint it.

## Anti-patterns
- rendering the entire world for a small tactical UI
- calculating visibility only from aesthetics, not game rules
- mixing “explored” with “currently visible” as if they were the same
- making all fog depend on a 60 fps RTT

## Strong recommendation
In most games, start with:
- simplified or prebaked base map
- separate fog overlay
- relevant icons
- updates by events, ticks, or sectors

Only increase visual complexity once tactical reading is solved.

## To expand later
- multiplayer with team-shared fog
- streaming persistent exploration
