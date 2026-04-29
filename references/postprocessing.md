# Postprocessing

## Goal
Use postprocessing in Three.js thoughtfully, understanding that it is an additional render chain and not a free decoration.

## Main rule
Postprocessing must justify its cost.

If the game already works visually without it, better. Adding it later is healthier than building the whole visual identity on an expensive, fragile chain from minute one.

## Technical base
Three.js sets up postprocessing with `EffectComposer` and a chain of passes.

Base pattern:
- `EffectComposer`
- `RenderPass`
- specific passes such as bloom or others
- `OutputPass` at the end

## What it really implies
The manual leaves an important idea: the composer works with intermediate render targets and chains passes together. That means:
- more memory
- more work per frame
- more places where something can go wrong

This also means postprocessing cost should be part of quality tiers, not remain a fixed and blind decision.

For custom render targets outside the post chain, see `render-targets.md`.

## Recommended default
- do not enable postprocessing by default in gameplay prototypes
- introduce it once the main loop is clear
- keep it modular and easy to turn off
- treat it as a quality preset if targeting mobile

## Resize
If there is a composer, resizing only the renderer is not enough.

Mandatory rule:
- update camera
- `renderer.setSize(...)`
- `composer.setSize(...)`

## Delta time
`composer.render(deltaTime)` may need the delta if some passes are animated.

Do not assume changing from `renderer.render()` to `composer.render()` is a dumb replacement with no consequences.

## Bloom and similar effects
Examples and the manual leave a practical conclusion:
- bloom can look beautiful
- bloom can also muddy the image and cost a lot
- it should not become makeup for weak visual direction

## Order and judgment
Useful questions before adding a pass:
- does it truly improve readability or tone?
- how much does it cost?
- can it be disabled by preset?
- does it hurt clarity on mobile or small screens?

And also:
- does it introduce hitches when activated or resized?
- does it need warmup or preparation before appearing at a critical moment?

## Shader passes
If something very specific is needed, `ShaderPass` allows custom effects.

Healthy rule:
- start with existing, small effects
- read the pass code before adopting it blindly
- keep important uniforms well localized and documented

## Anti-patterns
- adding bloom just because
- chaining many passes from day 1
- forgetting `composer.setSize()` on resize
- having no way to disable effects
- using postprocessing to hide material, lighting, or art-direction problems

## Strong recommendation
In web games, the best postprocessing is usually the minimum that gives identity without destroying performance or clarity.

For coordinated quality presets affecting passes, targets, and resolution, see `quality-tiers.md`.

## To expand
- recommended vs dangerous passes
- reduced resolution for certain effects
- mobile and profiling integration
