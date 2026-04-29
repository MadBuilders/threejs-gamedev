# Quality Tiers

## Goal
Design real quality presets for Three.js games, especially when there is postprocessing, render targets, and costs that vary heavily by device.

## Main rule
**Quality must be scalable by system, not just by one global switch.**

Think of tiers as a coordinated policy over:
- effective resolution
- shadows
- passes
- render targets
- draw distance
- world density
- asset variants if applicable

## Recommended default
Have at least:
- low
- medium
- high

If the project is serious or targets mobile, this stops being a luxury pretty quickly.

## What usually scales best
### Resolution
- `renderer.setPixelRatio()` with reasonable limits
- internal resolution of certain effects
- composer size and auxiliary render targets

### Postprocessing
- enable or disable passes
- lower bloom quality
- reduce blur resolution
- disable DOF on modest tiers

### Shadows
- on/off
- shadow map resolution
- number of shadow-casting lights
- useful shadow distance

### World
- prop density
- draw distance
- particle count
- frequency of certain secondary systems

## Render targets
The postprocessing manual leaves a key idea:
- `EffectComposer` already uses internal render targets
- some passes create more targets or their own buffers

That means the tier should not think only in terms of “enable bloom”, but in terms of:
- how many targets exist
- what resolution they live at
- whether all of them deserve full resolution

## Reduced resolution by effect
Very healthy pattern:
- not every effect needs full resolution

Especially:
- bloom
- blur
- some glow passes or similar combinations

Rule:
- if an effect tolerates half or lower resolution without breaking the image, use it as a preferred savings path

## Recommended vs dangerous passes
### More defensible
- moderate, measured bloom
- color grading or simple adjustments
- well-controlled output/tone mapping

### More dangerous
- depth of field in main gameplay
- long blur chains
- multiple costly passes at high resolution
- effects that degrade clarity on small screens

The DOF example is useful as a technical reference, but also as a reminder that something flashy can be quite expensive and does not always deserve to live in the playable loop.

## Tiering by postprocessing
Example of a reasonable policy:

### Low
- no DOF
- no bloom or minimal bloom
- essential output pass
- trimmed internal resolution

### Medium
- moderate bloom
- light color grading
- no very expensive persistent effects

### High
- justifiable full passes
- more polished bloom
- a premium effect if it truly adds value

## Runtime activation
Changing tier live can introduce stutter if:
- you create a new composer or render targets at a critical moment
- you recompile materials
- you resize large buffers without planning

Rule:
- prepare important changes outside sensitive moments
- if the change is strong, treat it as a system transition, not a trivial toggle

## Coherent presets
Do not create absurd tiers where:
- you lower shadows but leave expensive DOF on
- you lower pixel ratio but keep all premium post
- you save GPU but still have uncontrolled spawn and updates

Tiers must have internal coherence.

## Manual quality scaler first
Before thinking about complex auto-scaling:
- define good manual presets
- know what each tier turns off and keeps
- measure real scenes

Only then should you consider automatic adaptation if it is worth it.

The best way to validate whether a tier is well designed is to run it in repeatable stress scenes, not trust one pleasant scene.

For the automatic layer that decides when to lower or raise quality without thrash, see `adaptive-quality-scaling.md`.

## What to document per tier
- maximum pixel ratio
- active post passes
- size of special render targets
- shadows and their resolution
- draw distance
- prop/particle density
- visual notes and tradeoffs

This should also include the update frequency of non-critical targets such as minimaps, monitors, or remote cameras. See `render-targets.md`.

If the project uses mirrors, portals, or minimaps, treat them as distinct families inside the tier. See `render-target-families.md`.

## Anti-patterns
- one “low/high” button without knowing what it does
- treating all effects as equally expensive
- keeping large render targets by default on mobile
- enabling DOF or premium chains in gameplay for posturing
- changing tier without measuring resize and reconfiguration spikes

## Strong recommendation
Create a `qualityManager` or equivalent that:
- knows the current tier
- applies coordinated changes
- can affect renderer, composer, shadows, and world density
- exposes clear debug

## To expand later
- adaptive quality based on frame-time spikes
- concrete presets by genre
- relationship with custom render targets outside postprocessing
