# Portal Recursion Control

## Goal
Keep portals visually convincing without letting cost or complexity explode when one portal sees another portal, or sees itself indirectly.

## Main rule
**Do not start with infinite recursion.**
The healthy default is a non-recursive portal, or very limited and explicit recursion.

## What makes recursion difficult
When portal A sees portal B, A’s image may require rendering B, which in turn may require A again.

That complicates:
- number of passes
- render order
- camera stability
- clipping
- accumulated resolution cost

## Recommended default
- maximum depth 0 or 1 at the start
- own target per important visible portal
- moderate resolution per portal
- disable recursion in low tiers or mobile

## Useful mental model
Think of each recursion level as another derived view, not as “the same scene again with no cost”.

If `depth=0`:
- you render only the view on the other side

If `depth=1`:
- that view can include an additional representation of the next portal

Each level adds cost and fragility.

## Healthy strategies
### 1. Hard depth cap
The most important one.

Conceptual example:
- high desktop: depth 1 or 2, carefully measured
- medium desktop: 1
- mobile or low tier: 0

### 2. Decreasing resolution by level
Not every recursive level needs the same resolution.

Useful pattern:
- main level: base portal resolution
- next level: 0.5x or less
- far levels: maybe not rendered at all

### 3. Content clipping
Each recursive render should try to see less world, not more.

Levers:
- layers
- proxies
- maximum distance
- exclude cosmetic details

### 4. Aggressive update policy
Do not recompute all recursion every time.

Options:
- only if the portal is visible and occupies enough area
- only if the camera or portal changed enough
- alternate updates between secondary portals

## Portal crossing vs portal view
It is important to separate:
- seeing a portal on screen
- physically crossing a portal

Crossing requires spatial, physics, and camera coherence.
Visual recursion is another layer and should not complicate crossing more than necessary.

## Typical risks
- shimmering or seams from incorrectly chained matrices
- accidental feedback if a portal is used in the wrong pass
- explosive cost with two large portals facing each other
- trusting a recursive demo without measuring real frame time

## Recommended minimum benchmark
Measure:
- one visible portal without recursion
- two visible portals
- two portals facing each other
- recursion depth 1 versus 0
- impact of lowering target resolution

## Integration with quality tiers
Expose at least:
- `portalEnabled`
- `portalResolutionScale`
- `portalMaxRecursionDepth`
- `portalUpdateRate`
- `portalContentMask`

If the problem is clipping the portal area better or controlling frame overdraw, see `portal-masking-stencil-scissor.md`.

## Honest fallbacks
If the budget is not enough:
- portal without recursion
- frozen or reduced-update portal
- simpler proxy in the background
- disable premium visual and keep only the mechanic

## Strong recommendation
Premium portal, yes, but with a clear ceiling:
- recursion cap
- resolution by level
- specific benchmarks
- kill switch by tier

## To expand later
- crossing with complex physics
- chained portals in large worlds
