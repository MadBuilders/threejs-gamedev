---
name: threejs-gamedev
description: Build and maintain web games with pure Three.js, focused on reusable architecture, gameplay systems, asset pipelines, controls, performance, mobile constraints, debugging, and production-minded implementation patterns. Use when creating, extending, reviewing, or fixing a Three.js game without React or React Three Fiber, especially for decisions about code organization, render loop structure, assets, input, physics integration, optimization, or practical game-engine-style patterns.
---

# Three.js Gamedev

Trabajar con Three.js puro. No mezclar React ni R3F salvo que el usuario lo pida explícitamente.

## Workflow

1. Si el usuario quiere empezar un juego nuevo, cerrar primero kickoff, stack y primer slice jugable.
2. Identificar el problema principal.
3. Leer solo las referencias necesarias.
4. Preferir patrones mantenibles antes que demo code.
5. Tratar docs/manual/examples/repo oficial como base canónica.
6. Usar DeepWiki para preguntas concretas sobre estructura o implementación del repo oficial cuando ayude.
7. Usar la búsqueda semántica del foro oficial (Discourse AI) para edge cases, dolores recurrentes o preguntas específicas del ecosistema.
8. Explicitar tradeoffs de rendimiento, móvil y complejidad cuando importen.

## Defaults

- Three.js puro como base.
- `glTF` como formato principal de assets 3D.
- `setAnimationLoop` como loop por defecto.
- Separar bootstrap, render, world/systems y gameplay cuando el proyecto lo pida.
- Mantener addons explícitos y minimizados.
- Diseñar primero para claridad, luego para optimización.

## Mapa de referencias

### Kickoff y defaults de proyecto
- `references/game-kickoff-planning.md` para preguntas iniciales, kickoff brief y primer slice jugable.
- `references/phased-game-workflow.md` para forzar fases, validar mecánica antes de polish y evitar scope explosion.
- `references/threejs-game-viability.md` para viabilidad general, límites sanos, ideas que encajan bien e inspiración con scope realista.
- `references/default-project-stack.md` para stack por defecto, estructura de carpetas, Rapier y criterio singleplayer-first.
- `references/default-content-sourcing.md` para fuentes opinionadas de assets, texturas y audio provisional.
- `references/project-agents-md.md` para usar `AGENTS.md` como memoria operativa por juego.

### Core y gameplay
- `references/architecture.md` para estructura, bootstrap, loop, resize y lifecycle.
- `references/assets.md` para formatos, loaders e importación.
- `references/gltf-pipeline.md` para export, carga coordinada, compresión e instanciación.
- `references/animation-systems.md` para clips, mixers, actions y blending.
- `references/animation-state-machines.md` para estados visuales, transiciones y one-shots.
- `references/character-locomotion.md` para player controllers, grounded state, cámara y locomotion state.
- `references/input-controls.md` para input abstraction, teclado, touch, gamepad y raycasting.
- `references/physics.md` para integración de motor físico y límites de responsabilidad.
- `references/world-generation.md` para streaming, chunking y contenido procedural.
- `references/resource-lifecycle.md` para ownership, limpieza y `dispose()`.

### Performance y validación
- `references/mobile-performance.md` para presupuestos y reducción de coste.
- `references/profiling-budgets.md` para frame time, draw calls y budgets reales.
- `references/gpu-vs-cpu-heuristics.md` para distinguir cuello visual, lógico, mixto o stutter.
- `references/frame-pacing-stutter.md` para picos, warmup y activación suave.
- `references/quality-tiers.md` para presets coherentes por dispositivo.
- `references/adaptive-quality-scaling.md` para histéresis, cooldown y `renderScale`.
- `references/stress-scenes-benchmarks.md` para benches internos y escenas de estrés.
- `references/benchmark-reporting.md`, `references/benchmark-diffs.md` y `references/benchmark-thresholds.md` para runs comparables, diffs y thresholds.

### Rendering, RTT y lighting
- `references/render-targets.md` para RTT como subsistema, resolución, frecuencia y lifecycle.
- `references/render-target-families.md` para mirrors, refractors, portals y minimaps.
- `references/portal-recursion.md` para profundidad, resolución por nivel y fallbacks.
- `references/portal-masking-stencil-scissor.md` para recorte de área, stencil y overdraw.
- `references/minimap-fog-of-war.md` para minimapas tácticos, visibilidad y explored state.
- `references/fog-mask-blending.md` para masks y blending de fog.
- `references/transparency-pitfalls.md` para sorting, depth, alpha test y decisiones sanas con materiales transparentes.
- `references/lights-shadows.md` para estrategia de iluminación y shadow maps.
- `references/postprocessing.md` para cadenas de effects, resize y criterio de uso.

### Debug y multiplayer
- `references/debugging.md` para helpers e inspección visual.
- `references/multiplayer.md` para arquitectura base de red, snapshots e interest management.
- `references/multiplayer-consistency-models.md` para rollback, lockstep e hit validation.
- `references/server-rewind-weapons.md` para rewind o lag compensation por arma.
- `references/anti-cheat-anomalies.md` para telemetría, scoring de sospecha y mitigaciones.

## Reglas de criterio

- No copiar la documentación oficial dentro de la skill.
- No presentar demo code como arquitectura de producción.
- Marcar qué es core, qué es addon y qué es doctrina de proyecto.
- Recomendar herramientas externas solo cuando añadan un pipeline claro.
- Si una decisión afecta móvil o rendimiento, explicitar el tradeoff.

## Fuentes base

- `threejs.org/docs`
- `threejs.org/manual`
- `threejs.org/examples`
- `github.com/mrdoob/three.js`
- DeepWiki sobre el repo oficial como ayuda puntual
- búsqueda semántica del foro oficial (`/discourse-ai/embeddings/semantic-search.json`) como ayuda puntual para problemas concretos

## Estado actual

**v1 sólida en borrador**. Mejorar por casos reales, no por expansión abstracta.
