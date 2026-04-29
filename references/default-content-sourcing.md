# Default Content Sourcing

## Goal
Keep a small set of asset and audio sources for prototypes and first versions, with a more opinionated policy than “we’ll look for something later.”

## Main rule
**Fast placeholder first, legal complexity later.**
Do not block the game because you want the perfect asset too early.

## Default by resource type
### 3D placeholder / quick prototype
- **Kenney** for simple, coherent packs
- **Poly Pizza** for quick low poly assets

### Textures and materials
- **ambientCG** as a fairly healthy source for materials and surfaces

### Lightweight production 3D models
- **Sketchfab** only with extreme care around license and attribution

### Temporary audio
- **Pixabay** for quick temporary music and SFX
- **Freesound** as a secondary source, with extra care because licenses and quality vary a lot

## Healthy usage policy
### Prototype
Prioritize:
- speed
- enough consistency
- clear licenses

### Lightweight production
Prioritize:
- visual consistency
- license traceability
- reducing chaotic style mixing

## What to avoid
- mixing twenty visual styles without criteria
- downloading assets without recording license or source
- depending on Sketchfab without reviewing concrete usage terms
- blocking the game startup while searching for the final asset

## Recommended minimum registry
Save per project:
- asset source
- license or usage condition
- whether it is a placeholder or final candidate
- whether it requires attribution

This can live in the project `AGENTS.md` or in a short associated inventory.

## Audio
Healthy default:
- quick placeholders first
- do not invest heavily in final music until the game loop is validated
- keep SFX clear even if they are temporary

## Recommended pipeline
1. cheap, legally clear placeholder
2. validate gameplay
3. replace only what truly deserves improvement

## Strong recommendation
If there is no very specific need yet:
- Kenney / Poly Pizza for visual prototyping
- ambientCG for materials
- Pixabay for temporary audio
- Sketchfab only with serious license control

## Related references
- `game-kickoff-planning.md`
- `project-agents-md.md`
