# Fog Mask Blending

## Goal
Represent fog of war or tactical visibility with masks and blending that are readable, cheap, and coherent with game state.

## Main rule
**The mask expresses tactical state, not free decoration.**
Choose blending for clarity before choosing it for a “pretty effect”.

## What it tries to solve
- distinguish visible, explored, and unknown
- blend fog overlay with base map without losing readability
- avoid expensive or confusing visual solutions

## Three base states
The healthy minimum is usually:
- `visibleNow`
- `explored`
- `unseen`

The mask or combination of masks should make this reading extremely clear.

## Useful models
### 1. Simple binary mask
- visible / not visible

Very cheap, but limited.
Useful in prototypes or games without exploration memory.

### 2. Visible + explored
The most useful model in many games.

Pattern:
- currently visible: almost clean
- explored but not visible: dimmed
- unseen: covered

### 3. Masks by team or layers
Useful in multiplayer or games with multiple revealers.

## Blending options
### Multiplicative / darken-like
Very useful for darkening non-visible zones.

Advantages:
- simple
- cheap
- clear reading

Risk:
- if it darkens too much, useful information from the base map is lost

### Classic alpha lerp
Blends a fog layer over the base map.

Advantages:
- fine control
- easy to tune by state

Risk:
- gray or washed out if done without judgment

### Soft color coding
Add a distinct tint for explored versus visible.

Advantage:
- quick tactical reading

Risk:
- too much color and visual noise

## Healthy defaults
- explored darker or desaturated, not totally black
- unseen clearly hidden
- visible with maximum readability
- smooth transitions only if they do not sacrifice clarity

## Edges and smoothing
Options:
- hard edge for highly abstract tactical games
- soft feather if the style calls for it
- moderate blur on the mask, not on the whole minimap

Rule:
- smoothing should help reading, not smear it.

## Where to apply the mask
### Option A: in the overlay shader/material
Good when:
- you already have a simple minimap pipeline
- you want continuous visual control

### Option B: composition between base map and visibility texture
Good when:
- you clearly separate tactical data and map rendering
- you want updates independent from fog state

## Multiplayer and teams
If fog is shared by team:
- aggregate visibility from several revealers
- serialize tactical state compactly
- do not depend on what a client says it has seen as the only source of truth

## Performance
The mask should be cheaper than rerendering the world.

Good levers:
- low-resolution tactical visibility texture
- update by sector/tick
- small, localized blur
- simple composition

## Anti-patterns
- confusing explored with currently visible
- using pretty effects that destroy contrast
- recalculating a global mask at 60 fps unnecessarily
- applying heavy blur to the entire tactical UI

## Strong recommendation
Start with a clear visual policy:
- unseen hidden
- explored dimmed
- visible clean

Then choose the cheapest blending that preserves that reading.

## To expand later
- concrete shader examples
- visibility textures by chunk
- blending for highly diegetic styles
