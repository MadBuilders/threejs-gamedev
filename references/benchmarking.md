# Benchmarking

## Goal
Turn performance measurements into reproducible runs, honest diffs, and actionable verdicts. Unify reporting, diffing, and thresholds.

## Main rule
**A benchmark without a comparable trail and project thresholds gets forgotten and lies.**
The useful chain is: *reproducible run → structured report → validated diff → verdict under project thresholds*.

## What it tries to solve
- honest before/after comparisons
- avoid “I think it was faster”
- detect regressions in averages, percentiles, and spikes separately
- create a shared language for reviewing PRs or large changes
- translate numeric deltas into decisions (accept, watch, block)

## What it does not need on day 1
- perfect CI
- device farm
- industrial tooling
- automatic visual snapshots

Consistent semi-automation is enough.

---

## 1. Reproducible runs

### Minimum viable level
A benchmark must be able to:
1. start a specific scene
2. lock a reproducible configuration
3. run warmup + measurement window
4. capture key metrics
5. save a readable result

### Warmup and measurement window
- warmup phase to absorb compilation, loading, and caches
- stable measurement phase
- separate final report
- measure startup separately if it matters

### Reproducibility
The more controlled it is, the more valuable it is:
- fixed seed if there is randomness
- same camera route or prerecorded path
- same input sequence
- same run duration
- same visual configuration

Without this, comparing runs is mud.

### Camera paths and action scripts
Useful patterns without sophistication:
- orbit for 10s
- enable high tier at t=5s
- spawn 100 props at t=8s
- change skin at t=12s

That gives the spikes real context.

### What to capture
Run context:
- benchmark name, date, commit
- device/browser if known
- effective resolution and pixel ratio
- active tier
- relevant toggles (shadows, post, density, RTT, etc.)

Minimum metrics:
- average frame time
- p95 and p99
- worst relevant spike
- draw calls
- triangles
- geometries/textures/programs

Optional depending on the benchmark:
- build time
- load time
- time until “asset ready to show smoothly”
- spawn/despawn time
- adaptive scaler latency

### Separate throughput from stutter
Two different readings:
- steady state: average, percentiles, draw calls, and average state
- critical events: spike when enabling an asset, changing tier, creating an RTT, entering a chunk

Do not flatten everything into one number.

### Output format
Readable summary (markdown or text):
- benchmark, config, key metrics, observations

Structured data (JSON) with stable fields for later diffs and charts:

```json
{
  "bench": "draw-call-stress",
  "scenario": "instanced-vs-naive",
  "variant": "INSTANCED",
  "tier": "medium",
  "resolution": { "width": 1600, "height": 900, "renderScale": 0.8 },
  "sample": {
    "warmupMs": 3000,
    "measureMs": 10000,
    "frameTimeAvgMs": 14.8,
    "frameTimeP95Ms": 18.9,
    "frameTimeMaxMs": 29.4
  },
  "rendererInfo": {
    "calls": 1,
    "triangles": 240000,
    "geometries": 12,
    "textures": 5,
    "programs": 3
  },
  "notes": ["stable", "no visible stutter"]
}
```

---

## 2. Diffs between runs

### Prior rule
**Do not compare runs if they are not truly comparable.**
Before looking at metrics, validate that context and configuration are equivalent, or that the difference is declared.

### Context before numbers
Validate:
- same benchmark and variant
- same tier
- same effective resolution and `renderScale`
- same relevant toggles
- same warmup/measurement duration
- same seed or camera path if applicable

If they do not match, mark as `not comparable` or `comparable with reservations`, and do not sell the result as definitive.

### Diff types
1. **Stable throughput**: average frame time, p95/p99, draw calls, triangles, geometries/textures/programs.
2. **Spikes/events**: worst spike, spike when enabling an asset, changing tier, creating composer or RTT, spawn/despawn.
3. **Adaptive behavior**: time to first downgrade, number of changes, thrash.

### Healthy reading order
1. Is there any context difference?
2. Did p95/p99 change?
3. Did the worst relevant spike change?
4. Did the average change?
5. Did memory or draw calls change?

Avoid obsessing over the average while stutter gets worse.

### Classification
- **clear improvement**
- **clear regression**
- **mixed / tradeoff**
- **inconclusive**
- **not comparable**

### Typical tradeoff cases
- the average drops but the worst spike rises
- p95 improves but live memory increases
- draw calls drop but build time gets worse
- high tier improves but low tier breaks

Say it that way. Do not force binary success/failure.

### Diff shape

```json
{
  "bench": "postprocessing-stress",
  "baseline": "run-2026-04-17-a",
  "candidate": "run-2026-04-17-b",
  "comparable": true,
  "classification": "mixed",
  "deltas": {
    "frameTimeAvgMs": -1.4,
    "frameTimeP95Ms": -2.1,
    "frameTimeMaxMs": 5.8,
    "renderCalls": 0,
    "textures": 2
  },
  "highlights": [
    "clear p95 improvement",
    "worst spike when enabling bloom gets worse",
    "texture memory rises slightly"
  ]
}
```

---

## 3. Per-project thresholds

### Rule
**Thresholds are not universal.**
They come from the genre, hardware target, frame budget, and what the project considers acceptable.

### Starting point
Before setting thresholds, make clear:
- frame rate target: 60fps (~16.7ms), 30fps (~33.3ms), or mixed
- target hardware: desktop, mobile, low-end
- critical scenes: main gameplay, combat, loading

### Three threshold layers
- **noise**: below this, the change is not considered significant
- **warning**: deserves attention and a comment
- **blocker**: breaks budget or policy; do not accept without a justified exception

### By category
1. **Stable throughput**: the average can tolerate more; p95 usually matters more; if p95 is close to the budget ceiling, tighten it.
2. **Spikes and events**: stricter thresholds than for the average. A new 20ms spike in critical gameplay is not acceptable even if the average barely moves.
3. **Resources and memory**: look not only at the absolute delta, but also at whether the project was already tight on mobile or low tiers.

### By benchmark and platform
Do not use the same thresholds for:
- synthetic draw-call benchmark vs gameplay slice
- high desktop vs mobile

Healthy structure:
- global project defaults
- overrides by benchmark or family
- overrides by platform or tier

### Conceptual example

```json
{
  "project": "my-threejs-game",
  "targets": {
    "frameTimeAvgWarnMs": 0.5,
    "frameTimeAvgBlockMs": 1.5,
    "frameTimeP95WarnMs": 1.0,
    "frameTimeP95BlockMs": 2.5,
    "frameTimeMaxWarnMs": 4.0,
    "frameTimeMaxBlockMs": 8.0,
    "renderCallsWarn": 20,
    "renderCallsBlock": 50
  }
}
```

These are not universal numbers. They only illustrate the shape.

### Final verdict
Combine diff + thresholds:
- **ok / noise**
- **watch**
- **serious regression**
- **blocking**
- **acceptable tradeoff**

---

## Minimum project infrastructure

Small, sufficient layer:
- `benchRunner`: accessible benchmark scenes (query param, debug menu, or dedicated route), externally reproducible config (seed, tier, variant, duration, density, `renderScale`), metrics collector (frame times, percentiles, `renderer.info`, events), text + JSON output, relevant event markers.
- `benchDiff`: validates comparability, computes deltas, groups by categories, marks mismatches, emits classification.
- `benchThresholds`: global defaults + overrides by benchmark and platform, applied to the diff.

Keep it close to the project code, not as loose notes.

## Integrations

### With profiling and budgets
- which benchmark breaks the budget?
- which tier fixes it?
- which change improves the average but worsens spikes?

### With adaptive quality
Also record:
- time to first downgrade
- number of downgrades/upgrades
- thrash
- whether percentiles improved or only the average

### With GPU vs CPU heuristics
If the benchmark changes a clean visual or logic lever, the report helps classify the bottleneck: visual/GPU-ish, logic/CPU-ish, mixed, stutter/load.

### With human review
The diff does not replace looking at context:
- is the visual change worth the cost?
- does the slowdown appear only in a rare scene or in the main one?
- is the benefit desktop-only while punishing mobile?

## Useful inspiration from examples
`webgl_instancing_performance`:
- comparable variants
- `console.time()` to measure build
- same scene, different strategy

## Anti-patterns
- eyeballing performance and saving nothing
- comparing runs with different configurations without saying so
- mixing warmup, loading, and steady-state into one number
- capturing only average FPS
- not recording the active tier or `renderScale`
- changing several strong variables at once
- celebrating average improvements while ignoring worse spikes
- applying the same thresholds to every benchmark
- blocking changes because of tiny noise
- invented thresholds that are not tied to the frame budget

## Strong recommendation
A serious project benchmark always emits:
- a human summary with verdict
- comparable JSON
- a classification under the project thresholds

## Related references
- `stress-scenes-benchmarks.md`
- `profiling-budgets.md`
- `gpu-vs-cpu-heuristics.md`
- `adaptive-quality-scaling.md`
- `quality-tiers.md`
