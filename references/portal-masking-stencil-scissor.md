# Portal Masking with Stencil and Scissor

## Goal
Clip the portal view to the correct frame and avoid unnecessary overdraw, using stencil or scissor when the case justifies it.

## Main rule
**Not every portal needs stencil.**
And using stencil or scissor without understanding what problem they solve is a recipe for infernal render order.

## What it tries to solve
- prevent the portal view from spilling outside the frame
- reduce useless rendering outside the portal area
- support non-trivial frames or more precise clipping

## Two different tools
### Scissor test
Fits well when:
- the portal occupies a rectangle or screen area that can be approximated
- you want quick screen-space clipping
- you want simple fillrate savings

Advantages:
- mentally simple
- useful for limiting the cost of the portal pass

Limits:
- rectangular clipping
- less useful if the frame is irregular or transformed in a complex way

### Stencil buffer
Fits when:
- the portal frame has a more precise shape
- you need finer masking
- there is more serious portal composition with the scene

Advantages:
- more exact mask
- works better for non-rectangular or more controlled frames

Cost:
- more delicate render order
- easier to break GL state

## Recommended default
- start without stencil if the portal already works and the frame is simple
- add scissor first if the problem is overdraw or rectangular clipping
- reserve stencil for complex frames or composition that truly asks for it

## Conceptual pattern with scissor
1. project portal bounds to screen-space
2. calculate clipping rectangle
3. enable scissor for the portal pass
4. render only that region
5. restore state

Good use:
- small portal on screen
- several portals where you want to save fillrate

## Conceptual pattern with stencil
1. render the frame mask to stencil
2. configure tests so the portal view only paints where the mask allows
3. render the portal scene with that state
4. clear or restore stencil according to the pipeline

Good use:
- irregular frames
- surfaces where the portal must respect precise geometry

## Typical risks
- renderer state leaking between passes
- relying on stencil without clear cleanup
- using scissor with badly calculated bounds and clipping too much
- solving with stencil what really only needed a better portal quad or better clip

## Integration with recursion
Stencil or scissor do not eliminate the base cost of recursion.
They only help control where it is drawn.

You still need:
- depth cap
- resolution by level
- update policy

## Integration with quality tiers
Expose if needed:
- `portalUseScissor`
- `portalUseStencil`
- `portalMaskQuality`

In low tiers, often enough:
- no stencil
- simple scissor, or even neither if the portal is already small

## Useful debug
Look at:
- real portal area on screen
- apparent fillrate
- overdraw if tooling exists
- glitches when changing render order
- cost with and without scissor/stencil

## Anti-patterns
- enabling stencil by default on all portals
- not restoring GL/renderer state
- using scissor for very complex frames as if it were a perfect mask
- confusing visual mask with a complete solution for the portal system

## Strong recommendation
Think this way:
- rectangular screen-area problem: try scissor
- precise frame-shape problem: consider stencil
- global cost problem: first go back to resolution, recursion, and content

## To expand later
- curved or arbitrary frames
- interaction with postprocessing
- stencil in chains of multiple portals
