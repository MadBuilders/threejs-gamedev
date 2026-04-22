---
name: threejs-gamedev
description: Build, extend and review web games with pure Three.js (no React or R3F). Covers architecture, render loop, assets, input, physics integration, rendering/RTT, performance and mobile trade-offs, audio, UI/HUD, cameras, shaders, AI/navigation, persistence, build/deploy and debugging. Use for singleplayer-first 3D/2.5D web games where control and clarity matter more than framework convenience. Not for React Three Fiber projects.
---

# Three.js Gamedev

Trabajar con Three.js puro. No mezclar React ni R3F salvo que el usuario lo pida explícitamente.

## Workflow

1. Si el usuario quiere empezar un juego nuevo, cerrar primero kickoff, stack y primer slice jugable.
2. Identificar el problema principal.
3. Leer solo las referencias necesarias (ver *Uso del contexto*).
4. Preferir patrones mantenibles antes que demo code.
5. Tratar docs/manual/examples/repo oficial como base canónica.
6. Usar DeepWiki para preguntas concretas sobre estructura o implementación del repo oficial cuando ayude.
7. Usar la búsqueda semántica del foro oficial (Discourse AI) para edge cases, dolores recurrentes o preguntas específicas del ecosistema.
8. Explicitar tradeoffs de rendimiento, móvil y complejidad cuando importen.

## Uso del contexto

La skill tiene muchas referencias. Cargarlas todas en un turno es un anti-patrón.

Reglas duras:
- **Máximo 3 referencias por turno** salvo justificación clara.
- **Nunca leer el bloque avanzado de multiplayer si el proyecto es singleplayer**.
- Si el usuario pregunta algo transversal, leer primero el router de abajo y elegir; si sigue sin estar claro, preguntar antes de leer.
- Si una referencia remite a otra, no encadenar lecturas sin criterio: evaluar si la segunda de verdad cambia la respuesta.

## Router rápido

Intención del usuario → referencia por la que empezar.

- *"Quiero empezar un juego"* → `game-kickoff-planning.md`, luego `default-project-stack.md`.
- *"¿Cómo organizo el código?"* → `architecture.md`.
- *"¿En qué orden ataco el proyecto?"* → `phased-game-workflow.md`.
- *"Carga de modelos/texturas/audio"* → `assets.md`, y si hay 3D complejo `gltf-pipeline.md` (incluye **gltf-transform** para inspección/optimización de GLB).
- *"Texturas: maps, color space, tiling, compresión"* → `texturing-pipeline.md` (incluye **ribbon meshes sobre curvas** para caminos/ríos).
- *"El cielo se ve feo / materiales PBR apagados"* → `lights-shadows.md` sección **IBL con HDRI**.
- *"Cargar GLBs/HDRIs sin bloquear el boot"* → `assets.md` sección **placeholder first, swap later** (+ disposal correcto).
- *"Animaciones de personaje"* → `animation-systems.md` + `animation-state-machines.md`.
- *"Mover al jugador / cámara de seguimiento"* → `character-locomotion.md` + `cameras.md`.
- *"Tank / sin strafe, girar con A-D"* → `character-locomotion.md` sección **tank / vehicle-lite**.
- *"Mirar alrededor con el ratón sin que la cámara rompa el control"* → `cameras.md` (**seguimiento + offset**) + `input-controls.md` (**hold-to-look / pointer capture**).
- *"Minimapa o radar sin otro render pass"* → `ui-hud.md` (**Canvas 2D**), no `render-targets.md` salvo que necesites ver el mundo texturizado.
- *"Input (teclado, touch, gamepad)"* → `input-controls.md`.
- *"Necesito física"* → `physics.md`.
- *"¿Cómo hago colisiones sin motor físico?"* → `physics.md` sección **Antes de meter un motor** (escalera raycast → AABB/cápsula a mano → BVH → motor).
- *"¿Rapier mete overhead? / ¿lo meto ya?"* → `physics.md` secciones **Cuándo sí vale la pena el switch**, **Rendimiento** (coste concreto) y **Anti-patrones específicos de Rapier**.
- *"Mundo grande / streaming / proceduralismo"* → `world-generation.md`.
- *"Horizon feo / quiero relieve real en el terreno"* → `world-generation.md` + aplicar **Patrones de producción** de abajo (**heightfield / grid terrain**, **terrain como sistema**).
- *"Horizonte vacío pero el gameplay es plano"* → `world-generation.md` sección **Relieve de horizonte como silueta** (no meter heightfield completo para esto).
- *"Cientos de árboles/props iguales van lentos"* → `gltf-pipeline.md` sección **Migración de `clone()` a `InstancedMesh` por leaf**.
- *"Pocos árboles/props pero el juego va lento igual / apagar una categoría decor mejora mucho"* → `gpu-vs-cpu-heuristics.md` sección **"Pocos assets" no implica "barato"** (coste real = tris × doubleSided × PBR × shadow × worldScale²).
- *"Asset generado por IA (Meshy, Rodin, Luma, etc.), ¿qué reviso antes de meterlo?"* → `gltf-pipeline.md` sección **AI-generated GLBs** (doubleSided, texturas 2K, polycount, pipeline post-download, FrontSide en loader, no pasar skinned por AI-remesh).
- *"Sombras caras o con poca nitidez cerca del jugador"* → `lights-shadows.md` sección **Shadow camera que sigue al foco**.
- *"¿receiveShadow en foliage/árboles lo dejo?"* → `lights-shadows.md` sección **`receiveShadow` también cuesta por fragmento** (por defecto `false` en canopies y scattered trees).
- *"Stutter rítmico sin FPS obvio bajo"* → `profiling-budgets.md` (bullet de **asignaciones per-frame en el hot path**).
- *"Quiero source editable y runtime rápido"* → aplicar **Patrones de producción** de abajo y leer `assets.md` / `world-generation.md` según el cuello.
- *"Quiero editor de niveles / mapa authored"* → `level-editor-in-browser.md` (+ **Level editors / authored worlds** de abajo para doctrina).
- *"Limpiar recursos / memory leak"* → `resource-lifecycle.md`.
- *"Va lento en móvil"* → `mobile-performance.md` + `profiling-budgets.md`.
- *"¿Es GPU, CPU o stutter?"* → `gpu-vs-cpu-heuristics.md` + `frame-pacing-stutter.md`.
- *"Animaciones distorsionan el mesh (scale de hueso raíz, root motion no deseado)"* → `gltf-pipeline.md` sección **tracks de scale**.
- *"Quality settings y escalado"* → `quality-tiers.md` + `adaptive-quality-scaling.md`.
- *"Benchmark y regresiones"* → `benchmarking.md` + `stress-scenes-benchmarks.md`.
- *"Luces y sombras"* → `lights-shadows.md`.
- *"Espejos, portales, minimapas, agua"* → `render-targets.md` + `render-target-families.md` (+ `portal-*` o `minimap-fog-of-war.md` según caso).
- *"Transparencias que se ven raras"* → `transparency-pitfalls.md`.
- *"Postpro (bloom, SSAO, etc.)"* → `postprocessing.md`.
- *"Audio del juego"* → `audio-systems.md`.
- *"HUD, menús, overlays"* → `ui-hud.md`.
- *"Shader custom (dissolve, water, terrain blend, etc.)"* → `custom-shaders.md`.
- *"Pathfinding / enemigos / IA"* → `ai-navigation.md`.
- *"Guardar partida, progreso, settings"* → `persistence-save.md`.
- *"Build, compresión de assets, deploy"* → `build-deploy.md`.
- *"Debug visual"* → `debugging.md`.
- *"Multiplayer"* → ir a *Bloque avanzado* (bajo demanda explícita).

## Defaults

- Three.js puro como base.
- `glTF` como formato principal de assets 3D.
- `setAnimationLoop` como loop por defecto.
- Separar bootstrap, render, world/systems y gameplay cuando el proyecto lo pida.
- Mantener addons explícitos y minimizados.
- Diseñar primero para claridad, luego para optimización.
- **Singleplayer first** salvo requisito claro de multiplayer.

## Patrones de producción

- **Source editable -> artifact de runtime**: cuando un mundo/nivel crece, separar el archivo cómodo de editar (JSON, curvas, placements) del archivo cómodo de cargar en juego. Mantener fallback al source mientras el pipeline madura.
- **Worker-first para trabajo pesado**: terreno, chunking, máscaras, nubes o preprocesos largos no deberían competir con input/cámara/HUD en el hilo principal.
- **Authored data + LOD + instancing**: usar data authored para decidir *qué* va dónde; usar LOD e instancing para decidir *cómo* se dibuja barato. No mezclar layout con optimización ad-hoc.
- **Boot con fallbacks**: si falta un asset o artifact secundario, el juego debería degradar con una ruta más simple en vez de romper el arranque completo.
- **Heightfield / grid terrain para relieve real**: si el horizonte o el suelo necesitan forma de verdad, modelar el terreno como datos de altura (authorados, generados o mixtos) en vez de “parchear” un plano con fórmulas locales difíciles de razonar.
- **Heightfield no implica física**: un heightfield puede servir solo para render, solo para placement, o para render + colliders. No meter Rapier “porque hay colinas” si el juego solo necesita `getGroundHeight(x, z)` y quizá la normal del terreno.
- **Terreno como sistema, no como truco visual**: separar `terrain data` / `terrain build` / `terrain render` del resto de `level.ts`. Cuando el relieve importa, conviene que terreno, horizonte y posibles colliders salgan del mismo modelo mental.
- **Rapier cuando la interacción lo pide**: ruedas, suspensión, cuerpos rígidos apoyados en pendiente o contacto físico continuo sí empujan hacia Rapier/heightfield collider. Un walking game con relieve suave suele poder seguir kinemático.

## Level editors / authored worlds

- **El editor escribe source humano**: JSON o estructuras fáciles de inspeccionar/diffear. El editor no debería escribir formatos opacos como fuente principal.
- **Runtime loader separado del editor**: `levelDefinition` / schema por un lado, render/runtime por otro. Así el editor no arrastra dependencias del juego entero.
- **El bake es opcional y posterior**: primero validar que el workflow authored funciona. Solo añadir artifact/binario cuando el tamaño, el parseo o el boot lo justifiquen.
- **Fallback sano**: si el artifact derivado falta, cargar el source authored. Si eso falla, usar un default interno para no romper el arranque.
- **Data antes que render code**: paths, placements, spawn, boundary, props y tuning del layout deberían vivir en data editable, no enterrados en constantes visuales.
- **El editor es una ruta del mismo bundle**: activar con `?editor=1` y reusar bootstrap (renderer, scene, loaders) para que la preview *sea* el juego. Ver `level-editor-in-browser.md` para patrones concretos: save endpoint dev, doble `TransformControls` para translate+rotate simultáneos, snap+Shift, draft en `localStorage`, catálogo de assets extensible y disposal correcto de helpers.
- **Si el terreno importa, también debería authorarse**: para mundos con relieve real, tratar el terreno como data editable (height grid, masks, splines, stamps o mezcla) y no como una deformación escondida en el código de render del nivel.

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
- `references/texturing-pipeline.md` para maps, color space, tiling/anisotropy, compresión y blending de terreno.
- `references/animation-systems.md` para clips, mixers, actions y blending.
- `references/animation-state-machines.md` para estados visuales, transiciones y one-shots.
- `references/character-locomotion.md` para player controllers, grounded state, cámara, locomotion state y variantes (**tank controls** cuando el strafe compite con otra mecánica).
- `references/cameras.md` para follow cameras, spring-damped, orbital, cinematic y collision-aware.
- `references/input-controls.md` para input abstraction, teclado, touch, gamepad y raycasting.
- `references/physics.md` para integración de motor físico y límites de responsabilidad.
- `references/world-generation.md` para streaming, chunking y contenido procedural.
- `references/level-editor-in-browser.md` para meter un editor de niveles dentro de la misma app (save endpoint dev, dual `TransformControls`, snap, draft en `localStorage`, asset catalog, gotchas de disposal).
- `references/ai-navigation.md` para pathfinding, nav meshes, steering y behavior simple.
- `references/resource-lifecycle.md` para ownership, limpieza y `dispose()`.

### Presentación y UX
- `references/audio-systems.md` para buses, spatial audio, loading y pool de voces.
- `references/ui-hud.md` para HUD, menús, overlays DOM vs canvas y acoplamiento sano.
- `references/persistence-save.md` para guardar partida, progreso y settings.

### Performance y validación
- `references/mobile-performance.md` para presupuestos y reducción de coste.
- `references/profiling-budgets.md` para frame time, draw calls y budgets reales.
- `references/gpu-vs-cpu-heuristics.md` para distinguir cuello visual, lógico, mixto o stutter.
- `references/frame-pacing-stutter.md` para picos, warmup y activación suave.
- `references/quality-tiers.md` para presets coherentes por dispositivo.
- `references/adaptive-quality-scaling.md` para histéresis, cooldown y `renderScale`.
- `references/stress-scenes-benchmarks.md` para benches internos y escenas de estrés.
- `references/benchmarking.md` para runs reproducibles, diffs, thresholds y clasificación final (reporting + diffs + thresholds unificados).

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
- `references/custom-shaders.md` para `ShaderMaterial`, `onBeforeCompile`, patrones comunes y anti-patrones.

### Debug y build
- `references/debugging.md` para helpers e inspección visual.
- `references/build-deploy.md` para Vite build, compresión de assets, cache busting y deploy.

### Bloque avanzado (solo bajo demanda explícita)
No cargar estas referencias por defecto. Entrar aquí solo si el usuario declara multiplayer como core del juego, o si va a añadirlo a un proyecto singleplayer existente.

- `references/multiplayer.md` para arquitectura base de red, snapshots, interest management y **stack concreto recomendado (Colyseus)** con sus gotchas en 0.17.
- `references/multiplayer-consistency-models.md` para rollback, lockstep e hit validation.
- `references/server-rewind-weapons.md` para rewind o lag compensation por arma.
- `references/anti-cheat-anomalies.md` para telemetría, scoring de sospecha y mitigaciones.

Default sano para juegos casual / cooperativo / competitivo ligero: empezar por `multiplayer.md` y plantear Colyseus con monorepo (`client/` + `server/`). Saltar a los otros tres solo si el género lo justifica.

## Reglas de criterio

- No copiar la documentación oficial dentro de la skill.
- No presentar demo code como arquitectura de producción.
- Marcar qué es core, qué es addon y qué es doctrina de proyecto.
- Recomendar herramientas externas solo cuando añadan un pipeline claro.
- Si una decisión afecta móvil o rendimiento, explicitar el tradeoff.
- Si una referencia declara un anti-patrón, no contradecirlo sin justificar por qué este caso es excepción.

## Fuentes base

- `threejs.org/docs`
- `threejs.org/manual`
- `threejs.org/examples`
- `github.com/mrdoob/three.js`
- DeepWiki sobre el repo oficial como ayuda puntual
- búsqueda semántica del foro oficial (`/discourse-ai/embeddings/semantic-search.json`) como ayuda puntual para problemas concretos

## Estado actual

**v1.11**. Reforzada la doctrina de colisión/física a partir de preguntas frecuentes de kickoff ("¿cómo funciona la colisión en Three.js?", "¿Rapier mete overhead si lo meto?"):

- `physics.md`: bloque nuevo al principio **Antes de meter un motor** con la afirmación explícita *"Three.js no tiene detección de colisiones"* y la **escalera de cuatro estrategias** (raycast puntual → bounding volumes manuales → BVH sobre mesh estático → motor físico). Da permiso explícito a quedarse en el escalón 2 (colisión a mano en XZ o AABB) para la mayoría de juegos casual/walking/multijugador ligero.
- `physics.md`: sección nueva **El collider nunca es el asset visual** como regla dura. El GLB se renderiza, el collider lo declaras tú (radio, cápsula, AABB). Complementa al aviso de `gltf-pipeline.md` sobre GLBs de IA con `doubleSided: true` y 2–3M tris.
- `physics.md`: sección nueva **Cuándo sí vale la pena el switch a motor físico** con checklist operativa (ruedas/suspensión, stacking, pendientes con contacto continuo, proyectiles físicos, joints/constraints, mecánicas emergentes). Reemplaza el genérico "cuándo usar física completa" con señales concretas. Cierra con el corolario *"Rapier cuando la interacción lo pide"*.
- `physics.md`: ampliada la sección **Rendimiento** con el **coste concreto de Rapier** (bundle 1–1.5 MB compat, init async, step según tipo de cuerpo, fixed timestep obligatorio, sync sin asignaciones, queries cacheables) y **números mentales** orientativos (<1 ms con decenas de kinemáticos, 2–5 ms con 500+ dinámicos, 10–20 ms con trimesh del nivel entero).
- `physics.md`: sección nueva **Anti-patrones específicos de Rapier** con los 8 tiros en el pie típicos (trimesh del mundo, dinámico por defecto, sleep roto, crear/destruir en hot path, delta variable al step, raycasts per-NPC, sync con asignaciones, collider = GLB).
- `SKILL.md`: dos entradas nuevas en el router (*"¿Cómo hago colisiones sin motor físico?"* y *"¿Rapier mete overhead? / ¿lo meto ya?"*) para que estas preguntas caigan directamente en las secciones nuevas de `physics.md` en vez de en la entrada genérica *"Necesito física"*.

**v1.10**. Recogidos aprendizajes de una sesión de optimización centrada en assets generados por IA en un juego 3D real:

- `gltf-pipeline.md`: sección nueva **AI-generated GLBs (Meshy y similares)** con los defaults silenciosos que traen (`doubleSided: true` en opacos, texturas 2K por slot PBR, polycount de 0.5M–3M tris, sin compresión), el pipeline post-download canónico (`resize` → `webp` → `meshopt`), la regla de **forzar `FrontSide` en el loader** en vez de corregir el GLB (robustez ante re-exports futuros), el aviso de **no pasar skinned meshes por AI-remesh** (rompe weights/skeleton/animaciones), y la práctica de **variant scale como dato authored, no baked in GLB** para poder regenerar la malla sin retunear la escena.
- `gpu-vs-cpu-heuristics.md`: sección nueva **"Pocos assets" no implica "barato"** con la fórmula mental del coste GPU real por asset (`tris × instancias × fragments × doubleSided × PBR samples × shadow samples × worldScale²`) y el aviso explícito de que **`InstancedMesh` colapsa draw calls pero no baja el coste por instancia**. Nuevo síntoma en GPU-bound: apagar una categoría decor entera gana mucho aunque solo haya 20 instancias.
- `lights-shadows.md`: sección nueva **`receiveShadow` también cuesta por fragmento** — no sólo `castShadow`. En foliage denso y canopies que están por encima del jugador, `receiveShadow = false` es casi siempre la respuesta correcta. Recomendación de exponer el flag en los factories de carga para que árboles y props compartan loader pero diverjan en política de sombras.
- `SKILL.md`: tres entradas nuevas en el router (assets de IA, pocos assets que tanquean, `receiveShadow` en foliage).

**v1.9**. Recogidos aprendizajes de una pasada de optimización sobre un juego single-player 3D ya en producción:

- `world-generation.md`: nueva sección **Relieve de horizonte como silueta** para el caso "el gameplay es plano pero el horizonte se ve vacío". Receta reutilizable basada en **máscara radial · patrón angular · detalle fbm**, con tradeoffs frente a heightfield authored y aviso explícito de que es render-only (no tocar colliders / navmesh con esto).
- `gltf-pipeline.md` (Caso 3): sección nueva **Migración de `clone()` a `InstancedMesh` por leaf** con la plantilla reutilizable de aplanar leaf meshes al cargar y exponer un factory `createInstancedMeshes(placements)` paralelo al `instance()` existente. Incluye reglas sobre agrupar por variante, mantener `instance()` para casos interactivos y qué pasa con obstáculos/colisión.
- `lights-shadows.md`: sección nueva **Shadow camera que sigue al foco** (sun position + `sun.target` moviéndose con el jugador manteniendo el offset original) como alternativa al frustum gigante estático. Permite bajar `mapSize` y combinar con `PCFShadowMap` sin perder nitidez cerca.
- `assets.md`: nueva sección **Prefetch paralelo para colecciones de props** con el patrón `Promise.all(uniqueKinds.map(getModel))` cuando el loader ya cachea por kind; elimina el serializado oculto del primer prop de cada tipo. Nota sobre pool limitado si los kinds únicos son cientos.
- `ui-hud.md` (minimap): bullet extra sobre **cull por distancia antes del píxel math** (`dx² + dz² > radio² · 2`) cuando la lista de obstáculos crece a cientos.
- `profiling-budgets.md`: el bullet CPU-bound sobre "trabajo JS evitable" se expande a **asignaciones per-frame en el hot path**, con la receta de scratch buffers a nivel de módulo y el gotcha de no reasignar arrays compartidos con `result.xxx` (usar `length = 0`).
- `SKILL.md`: cinco entradas nuevas en el router para cada uno de los patrones anteriores.

**v1.8**. Terreno/worldbuilding más explícito para juegos con relieve real:

- `SKILL.md`: router rápido con una entrada nueva para *horizon feo / quiero relieve real en el terreno*.
- `SKILL.md`: en **Patrones de producción** se añaden dos heurísticas nuevas: **heightfield / grid terrain** y **terrain como sistema**.
- `SKILL.md`: se aclara que **heightfield no implica Rapier** y que Rapier entra sobre todo cuando hay ruedas/suspensión o contacto físico continuo.
- `SKILL.md`: en **Level editors / authored worlds** se fija que, si el terreno importa, también debería authorarse como data editable y no vivir como deformación opaca en render code.

**v1.7**. Recogidos aprendizajes de montar un editor de niveles real dentro del bundle del juego:

- Nueva referencia `level-editor-in-browser.md` con patrones probados: editor como subruta `?editor=1` que reusa el bootstrap, save vía endpoint dev `POST /dev/level` con guard de `NODE_ENV`, `localStorage` draft con restore silencioso, **doble `TransformControls`** al mismo objeto (translate XZ + rotate Y simultáneos) para evitar el toggle move/rotate, snap 0.5m / 15° con Shift para libre, catálogo de assets centralizado para extensibilidad sin tocar el editor, backward-compat del schema con campos opcionales + helpers `resolveXxx`, y la trampa clásica de `group.children` que no se vacía al hacer sólo `dispose()` (helpers fantasma).
- `SKILL.md`: router y bloque **Level editors / authored worlds** apuntan a la nueva referencia. Añadida doctrina de que **el editor es una ruta del mismo bundle**, no una app aparte.

**v1.6**. Authoring/editor pipeline más explícito:

- `SKILL.md`: nueva entrada de router para **editor de niveles / mapa authored**.
- Nuevo bloque **Level editors / authored worlds** para fijar el patrón reusable: source humano, loader separado, bake opcional, fallback al source y layout guardado como data.

**v1.5**. Patrones de producción reforzados a partir de juegos web ya enviados:

- `SKILL.md`: nuevo bloque **Patrones de producción** para recordar cuatro heurísticas que suelen llegar tarde pero conviene modelar pronto: **source editable -> artifact de runtime**, **worker-first heavy subsystems**, **authored data + LOD + instancing**, y **boot con fallbacks**.
- Router rápido: añadida una entrada explícita para cuando el problema es menos "cómo renderizo esto" y más "cómo paso de datos/editing a runtime sin romper el proyecto".

**v1.4**. Ajustes tras seguir iterando el mismo proyecto real hasta una fase más madura:

- `project-agents-md.md`: reforzado que `AGENTS.md` debe **refrescarse cuando el proyecto cambia de fase** y que conviene abrir documentos satélite (`MULTIPLAYER.md`, etc.) cuando un subsistema gana roadmap propio. Evita que la memoria operativa se quede congelada en el V0.
- `default-project-stack.md`: añadido patrón de evolución sana a **monorepo `client/` + `server/` + `shared/`** cuando multiplayer deja de ser hipotético, y nota sobre authoring/data-driven layout (`public/levels/` + definiciones explícitas) para no enterrar contenido jugable en constantes dispersas.
- `multiplayer.md`: nuevas reglas concretas sobre **validar score/progreso contra estado que el servidor ya conoce** (no contra payloads de claim) y sobre **no bloquear rondas por persistencia externa**; persistencia de leaderboard en background + timeout como default sano.

**v1.3**. Añadidos aprendizajes de un proyecto multijugador real (cliente Three.js puro + servidor Colyseus en monorepo):

- `multiplayer.md`: nueva sección **Stack concreto recomendado: Colyseus** con cuándo elegirlo, cuándo no, y los **gotchas de la 0.17** que cuestan tiempo (`MapSchema` no iterable, `getStateCallbacks` reemplaza a `onAdd`/`onRemove` directos, hidratación tardía del estado, `useDefineForClassFields: false`, `@types/express` para Express 5). También: patrón de integración con conexión no bloqueante, `MultiplayerHandle` único como capa de aislamiento, identidad visual determinista server-side (color hue desde paleta fija), y smoke test multi-cliente headless.
- `animation-systems.md`: sección **Gotchas concretos al clonar SkinnedMesh** (no vale `Object3D.clone`, exports nombrados de `SkeletonUtils.js`, materiales y geometrías compartidos por el clone, regla de ownership en `dispose`) + patrón **source + instance** (`loadCharacterSource` / `createCharacterInstance`) para reusar GLBs entre jugador local, NPCs y remotos sin refetch ni doble parseo. Anti-patrones extendidos.

**v1.2**. Patrones genéricos probados en producción web:
- `cameras.md`: **follow detrás + yaw/pitch offset con decay** (visuales desacoplados del frame de movimiento cuando otra mecánica fija la referencia).
- `input-controls.md`: **hold-to-look** con `pointerdown` + `setPointerCapture` como alternativa a pointer lock.
- `character-locomotion.md`: **tank / vehicle-lite** (W/S eje, A/D giran; facing como estado, no derivado de velocidad).
- `ui-hud.md`: **minimapa/radar con Canvas 2D**, player-up.
- `gltf-pipeline.md`: **tracks de scale** en clips que distorsionan rigs retargeteados + sección **gltf-transform (CLI)**.
- `texturing-pipeline.md` nuevo: maps, color space, tiling, compresión y **ribbon meshes sobre curvas** para caminos/ríos.
- `lights-shadows.md`: **IBL con HDRI** (`PMREMGenerator` como `scene.environment` + `scene.background`).
- `assets.md`: **placeholder first, swap later** y recetas de `dispose()` al sustituir material/textura/render-target.

Router actualizado. El bloque avanzado de multiplayer ya tiene un default concreto (Colyseus) además de la doctrina general; sigue cargándose solo bajo demanda.
