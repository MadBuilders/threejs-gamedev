# Adaptive Quality Scaling

## Goal
Adjust visual quality at runtime in a stable, controlled way to protect frame time and prevent the game from turning into a festival of stutter or erratic changes.

## Main rule
**Adaptive quality must not react to one isolated bad frame.**
It must respond to trends, sustained spikes, and real context.

## What it tries to solve
- GPU overloaded on certain devices
- scenes whose cost changes a lot
- spikes from postprocessing, shadows, or internal resolution
- need to preserve a stable feel without forcing the player to open options

## What it does not try to solve
- disastrous CPU logic
- broken lifecycle
- stutter from compilation or asset activation
- badly designed benchmarks

If the real problem is CPU, recompiling shaders, or poorly distributed mass spawn, lowering resolution can hide it but not cure it.

For a prior layer of practical diagnosis before deciding which lever to touch, see `gpu-vs-cpu-heuristics.md`.

## Healthy base: manual presets first
Before adapting anything automatically, you need coherent manual tiers:
- low
- medium
- high

Adaptive quality should move between good presets or modify a very controlled subset of variables.

## Best levers for automatic adaptation
### 1. Internal resolution
Usually the cleanest lever when the bottleneck is GPU.

Strong pattern derived from the `responsive` manual:
- compute the real drawing-buffer size explicitly
- avoid opaque magic when the real resolution matters
- be able to cap the maximum internal pixel count

This favors a variable such as:
- `renderScale` between 0.6 and 1.0, for example

## 2. Resolution of certain effects
Very good second lever:
- half-resolution bloom
- reduced blur
- smaller auxiliary targets

The same applies to minimaps, monitors, and other non-critical RTTs: it is often better to lower their size or frequency than to degrade the main image first. See `render-targets.md`.

## 3. Premium postprocessing
Good lever, but with higher stutter risk:
- DOF off
- cheaper bloom
- disable premium chains

## 4. Shadows
Useful, but visually more delicate:
- lower shadow map resolution
- reduce distance
- limit shadow-casting lights

## 5. Secondary density
Only for non-critical systems:
- particles
- decorative props
- secondary vegetation

## Levers to touch less often
- abrupt material changes that recompile shaders
- frequent destruction and recreation of the full composer
- aggressive toggling of visible features every few seconds
- changes that affect readability or main gameplay

## Correct input signal
Do not use only instantaneous FPS.
Use better signals:
- smoothed frame time
- percentiles or recent window
- repeated spikes
- system context

Reasonable pattern:
- maintain moving average or EMA of frame time
- keep a counter of consecutive bad frames
- detect long spikes, not just isolated accidents

## Hysteresis or you eat hell
Without hysteresis, the scaler moves up and down like a drunk.

Rule:
- downgrade relatively quickly when degradation is sustained
- upgrade slowly only if there is comfortable margin for long enough

Conceptual example:
- downgrade if frame time exceeds the target for N frames or X accumulated ms
- upgrade only after several seconds of comfortable stability

## Cooldown
After applying a change, wait.

Without cooldown:
- you do not know which change helped
- you chain resizes
- you create more stutter than you were trying to avoid

## Separate intervention levels
### Level 1, fine adjustment
- lower `renderScale`
- reduce internal resolution of effects

### Level 2, moderate trimming
- cheaper bloom
- smaller shadows

### Level 3, tier change
- move from high to medium
- move from medium to low

This avoids turning off half the game because of a brief drop.

## Strong recommendation on resolution
Prefer an explicit drawing-buffer sizing policy over depending blindly on `renderer.setPixelRatio()`.

Strong reason from the `responsive` manual:
- knowing the actual buffer size matters a lot in postprocessing, shaders, screenshots, picking, and render targets
- it is also useful to cap maximum pixels

## Scaling by pixel budget
Very defensible pattern:
- define a maximum internal pixel count by tier or device
- if the real size exceeds that budget, apply `renderScale`

This is especially valuable on mobile and high-DPI displays.

## Changes at safe moments
Do not apply large changes in any random frame just because.

Better moments:
- pause
- menu
- transition
- fade
- after an encounter
- when the player is not in a critical maneuver

If the change happens during live gameplay, keep it small and low-visibility.

## Integration with frame pacing
A good adaptive scaler protects regularity, not just the average.

That is why you should watch:
- recent spikes
- resize cost
- target recreation
- quality manager changes

If the remedy causes stutter when applying changes, redesign the transition.

## Integration with quality tiers
Healthy pattern:
- `qualityManager` defines coherent presets
- `adaptiveScaler` decides whether to move within a margin or drop tier

Separating responsibilities helps a lot.

## Integration with benchmarks
Do not validate adaptive quality in one pleasant scene.
Test it in:
- postprocessing stress
- asset activation stress
- gameplay slice benchmark
- spawn/chunk benchmark

This shows whether it:
- truly stabilizes
- reacts too late
- changes too often
- breaks visual clarity

## Recommended debug
Expose at least:
- current tier
- `renderScale`
- smoothed frame time
- recent important spikes
- remaining cooldown
- reason for the last downgrade/upgrade

## Anti-patterns
- using instantaneous FPS as the only signal
- lowering quality because of one isolated spike
- moving up and down without hysteresis
- touching too many variables at once
- using adaptive quality to hide broken CPU or lifecycle
- applying large postprocessing changes in the middle of sensitive gameplay

## Strong recommendation
Create two separate layers:
- `qualityManager` for presets and coordinated changes
- `adaptiveScaler` for observation, hysteresis, cooldown, and decisions

## To expand later
- concrete heuristics by genre
- explicit separation between GPU-bound and CPU-bound
- upscale/downgrade policies with percentiles
- integration with telemetry and reporting
