# Physics

## Objetivo
Integrar física en juegos Three.js de forma pragmática, evitando que el motor físico se coma la arquitectura o complique el gameplay más de la cuenta.

## Antes de meter un motor

Three.js **no tiene detección de colisiones**. Es una librería de render. Lo que ofrece son helpers geométricos (`Raycaster`, `Box3`, `Sphere`, `Plane`, `Frustum`) con sus métodos `intersect*`, que son matemáticas, no un sistema de colisión.

Eso significa que la primera decisión no es "qué motor físico uso", sino **"¿necesito un motor?"**. En la mayoría de juegos casual/3D ligeros la respuesta es no.

Escalera de estrategias, de menor a mayor coste:

1. **Raycasting puntual**. `THREE.Raycaster` contra un mesh o lista de meshes. Ideal para picking (click), line-of-sight, altura de terreno debajo del player, hit tests de armas. Barato si los targets son pocos o el mesh está acelerado con BVH.
2. **Bounding volumes manuales**. Lista propia de colliders (`{ center, radius }` para cilindros/esferas en XZ, `Box3` para AABB). Tests a mano por frame (`dx*dx + dz*dz < r*r`, o `Box3.intersectsBox`). Suficiente para el 80% de juegos casual: walking games, top-down, shooters ligeros, multijugador simple. Sin WASM, sin step, sin broadphase — solo matemáticas.
3. **BVH sobre mesh estático**. Con [`three-mesh-bvh`](https://github.com/gkjohnson/three-mesh-bvh) cuando sí necesitas queries contra geometría compleja del mundo (raycast contra paredes de un nivel tallado, shapecasts para movimiento preciso). Sub-ms para geometrías grandes.
4. **Motor físico real** (Rapier, Cannon-es, Ammo). Cuando hay **fuerzas, contactos persistentes, resolución automática entre cuerpos** o constraints. Entonces sí merece la pena pagar el coste.

Elegir el escalón más bajo que resuelve el problema. Subir de escalón solo cuando el anterior se queda corto, no por inercia.

## El collider nunca es el asset visual

Regla dura: el mesh del GLB (el modelo bonito con miles de tris, normales, PBR, skinning) **no** es el collider. Confundirlos es un anti-patrón clásico y caro.

Razones:
- Colisión contra geometría arbitraria es carísima comparada con una cápsula o AABB.
- El mesh puede traer huecos, normales raras, `doubleSided: true` silencioso (muy común en GLBs generados por IA), triángulos degenerados. Todo eso produce bugs de colisión difíciles de debuggear.
- Un artista o tool externa puede re-exportar el GLB con más detalle y de repente tu colisión se rompe o se encarece sin avisar.
- El runtime necesita una representación **estable y predecible**: un radio, una cápsula, una caja. Eso no lo da el GLB por sí solo.

Pipeline correcto:
- El **asset visual** vive en el GLB.
- El **collider lógico** lo declaras tú (radio de tronco para árboles, AABB para casas, cápsula para personajes). Puede ser un número hard-codeado por variante, un valor calculado una vez desde el bounding box del mesh al cargarlo, o un collider authoreado en el editor de niveles.
- Ambos comparten transform (position/rotation/scale) pero son representaciones distintas.

Esto aplica a los cuatro escalones de la sección anterior. Vale tanto si haces colisión a mano como si metes Rapier: **nunca** pasas el trimesh del GLB como collider por defecto.

## Regla principal
Usar física solo donde aporte valor real.

No todo objeto necesita simulación completa. Muchas veces basta con:
- colisiones simples
- triggers
- movimiento kinemático
- constraints puntuales

## Separación sana
Three.js renderiza. El motor físico simula.

No mezclar ambas responsabilidades.

Pensar en tres capas:
1. **representación visual**
2. **representación física**
3. **sincronización entre ambas**

## Recomendación inicial
Tener una capa o bridge de physics que:
- cree cuerpos y colliders
- avance la simulación
- sincronice transforms visuales
- exponga eventos o consultas útiles al gameplay

## Cuándo usar física completa
Sí suele merecer la pena para:
- objetos que caen, rebotan o empujan de verdad
- interacción sistémica entre varios cuerpos
- vehículos o mecanismos si el juego depende de ello
- gameplay basado en equilibrio, fuerzas o colisiones emergentes

## Cuándo NO hace falta
A menudo no hace falta para:
- triggers simples
- pickups
- obstáculos estáticos
- puertas o plataformas con movimiento guionado
- detección básica de proximidad

## Cuándo sí vale la pena el switch a motor físico

Pregunta operativa para decidir meter Rapier (o equivalente) en un proyecto que hoy resuelve colisión a mano: **¿la mecánica necesita algo que no escribo en 50 líneas?**

Checklist concreta. Si aparece alguno de estos, Rapier deja de ser overhead y empieza a ser ahorro:

- **Vehículos con ruedas y suspensión**. El raycast vehicle controller es muy difícil de clonar a mano bien; Rapier lo trae.
- **Apilamiento de cuerpos** (cajas, barriles, torres que se derrumban). Resolver contactos múltiples a mano es un pozo sin fondo.
- **Pendientes reales con contacto continuo** donde el player tiene que pegar al suelo sin clipping en triángulos ni saltitos en bordes de mesh.
- **Proyectiles físicos con rebote, fricción y efectos emergentes** (no balas rectas — esas son raycast).
- **Joints y constraints** (puertas con bisagra, cadenas, puentes colgantes, ragdolls).
- **Mecánicas sistémicas emergentes** donde el diseño *depende* de que los cuerpos interactúen entre sí sin scripting (puzzles físicos, dominos, empujar objetos a zonas).

Señal contraria (probablemente no hace falta):
- Personajes que solo tienen que no atravesar obstáculos y chocar entre ellos → cilindros/cápsulas en XZ a mano.
- Pickups, triggers, zonas → AABB contra AABB, sin motor.
- Nivel plano o con relieve suave sin contacto físico nítido → `getGroundHeight(x, z)` y seguir kinemático.
- Disparos o line-of-sight → `Raycaster`, no cuerpos.

Corolario: **Rapier cuando la interacción lo pide**. No meterlo "por si acaso" o "porque todo juego 3D serio lleva física". Cada escalón de la escalera de estrategias tiene su nicho.

## Principio clave
No usar un motor físico para resolver un problema que ya entiendes mejor con lógica de juego.

La física realista no siempre produce el mejor juego.

## Tipos de cuerpos y uso práctico

### Estáticos
Para suelo, paredes, mundo fijo.

### Dinámicos
Para objetos que deben reaccionar a fuerzas y colisiones.

### Kinemáticos
Para personajes, plataformas o elementos gobernados por lógica propia pero que aún interactúan con el mundo físico.

## Colliders
Preferir colliders simples siempre que sea posible:
- cajas
- esferas
- cápsulas
- planos aproximados

No usar malla compleja como collider por defecto salvo necesidad real.

## Player controller
El personaje principal suele necesitar cuidado especial.

Recomendaciones:
- no depender totalmente de física cruda para el control del player
- separar intención de movimiento de respuesta física
- definir bien suelo, salto, pendiente y contacto
- evitar que el personaje se sienta "gelatinoso" por buscar realismo

Patrón fuerte que sale muy bien parado en examples reales:
- player kinemático
- collider simple, normalmente cápsula
- mundo consultado con octree, queries o colliders estáticos
- gravedad, grounded state y salto resueltos por el controller

Eso suele dar un resultado mucho más controlable que soltar un rigid body humanoide y rezar.

## Timing y update
La física necesita un ritmo claro.

Reglas útiles:
- usar step de simulación controlado
- no dejar que el delta variable rompa la estabilidad
- sincronizar visuales después del paso físico
- registrar claramente el orden de update

## Rendimiento
La física también consume bastante.

Vigilar:
- número de cuerpos activos
- número de colliders complejos
- frecuencia de consultas
- coste de colisiones continuas
- objetos dormidos que podrían no actualizarse

### Coste concreto de Rapier

Cuando se mete Rapier, el coste real se reparte así (orden de magnitud para tenerlo en la cabeza):

- **Bundle**. `@dimforge/rapier3d-compat` pesa ~1–1.5 MB. La versión `compat` inlinea el WASM en JS para evitar dolor con Vite/ESM. Se nota en el boot móvil.
- **Init async**. `await RAPIER.init()` antes de crear el world. Centenas de ms en móvil de gama baja. No es per-frame pero sí retrasa el "first playable". Arrancarlo en paralelo con la carga de assets cuando se pueda.
- **`world.step()` por frame**. Con decenas de cuerpos kinemáticos y colliders primitivos, sub-ms en desktop. Escala con:
    - cuerpos **dinámicos** activos (los kinemáticos y estáticos son casi gratis)
    - pares en contacto (el broadphase filtra el resto — no importa el total de colliders, importa cuántos están próximos)
    - tipo de collider: cuboid/ball/capsule son baratos; **trimesh y heightfield grandes** pesan mucho en queries y memoria
- **Sincronización Rapier ↔ Three.js**. Cada frame se leen `body.translation()` / `body.rotation()` y se vuelcan al `Object3D`. Mal hecho (objetos nuevos por frame, `.clone()` en el hot path) genera GC visible. Usar scratch vectors/quaternions a nivel de módulo.
- **Fixed timestep obligatorio**. Rapier espera `dt` fijo (1/60 típicamente). Pasarle el `dt` variable del `rAF` directo produce inestabilidad *y* más substeps = más coste. Patrón correcto: accumulator con fixed step, render interpolado aparte.
- **Queries (raycast, shapecast)**. Baratas por llamada, pero spamearlas cada frame por cada NPC suma. Compartir resultados y cachear cuando tiene sentido (ground check puede salir del step del propio body en vez de un raycast extra).

Números mentales (desktop decente, juego web singleplayer):

- 50–200 kinemáticos + puñado de dinámicos + colliders primitivos → **<1 ms/frame** de `world.step()`. Invisible.
- 500+ dinámicos interactuando → **2–5 ms**. Notable, aún asumible.
- Trimesh del nivel entero + raycasts per-NPC sin culling → **picos de 10–20 ms** y stutter. Eso es donde duele de verdad.

Conclusión práctica: Rapier bien usado es muy rápido. "Rapier va lento" casi siempre se traduce en "Rapier mal integrado" — ver anti-patrones específicos abajo.

## Gameplay y física
La física debe servir al diseño, no dominarlo.

Preguntas útiles:
- ¿esto mejora la sensación del juego?
- ¿es más divertido o solo más realista?
- ¿puedo conseguir un resultado mejor con lógica más simple?

## Anti-patrones
- meter física a todo por inercia
- usar colliders complejos para cualquier cosa
- acoplar gameplay directamente a callbacks del motor físico
- no distinguir entre cuerpo visual y cuerpo físico
- confiar en física realista para arreglar mal diseño de controles

## Anti-patrones específicos de Rapier

La mayoría de "Rapier va lento" o "Rapier da problemas raros" en proyectos reales se explica con uno de estos:

- **Trimesh collider del nivel entero**. Pasar el GLB del mundo tal cual a `ColliderDesc.trimesh` crea un collider con miles/decenas de miles de triángulos. Memoria alta, queries lentas, contactos impredecibles. Correcto: cuboides/heightfield para el grueso, trimesh solo para piezas puntuales que lo justifiquen (una roca tallada, una escultura que *necesita* precisión).
- **Cuerpos dinámicos donde bastaba kinemático**. Players y NPCs no necesitan ser dinámicos salvo que se quiera rebote físico entre ellos. Kinemático = tú mueves, Rapier detecta y opcionalmente te devuelve corrección. Dramáticamente más barato y más controlable.
- **Sleep roto**. Rapier duerme cuerpos quietos automáticamente y es casi gratis en esa fase. Fuerzas pequeñas aplicadas cada frame "para asegurar", transforms reseteados, o `setNextKinematicTranslation` incluso cuando no ha cambiado la posición, los despiertan y se pierde esa ventaja.
- **Crear/destruir colliders en el hot path**. Props reciclables, balas, partículas con colisión: pool siempre, activar/desactivar en vez de `world.createCollider` + `world.removeCollider` cada vez.
- **Delta variable al step**. Pasar el `dt` del `rAF` directo a `world.step()` en vez de fixed timestep con accumulator. Produce inestabilidad (explosiones, tunneling, jitter) y coste variable. Regla dura: el step es fijo, el render interpola.
- **Raycasts per-frame por NPC para ground check** cuando con el propio contacto del body o un sphere cast puntual basta. Multiplica por N el coste de queries.
- **Sync con asignaciones por frame**. `new THREE.Vector3(body.translation().x, ...)` por cada cuerpo y por frame produce GC visible en stutter. Scratch buffers a nivel de módulo y asignación directa (`obj.position.set(t.x, t.y, t.z)`).
- **Mismo collider para render y simulación**. Ya cubierto en *El collider nunca es el asset visual*, pero con Rapier duele especialmente porque el motor sí te deja hacerlo y no avisa del coste.

## Pendiente de ampliar
- elección concreta de motor recomendado
- patrones para personajes y equilibrio
- triggers y sensores
- rollback o reconciliación si hay multiplayer
- herramientas de debug de colliders y contactos
- integración con world generation y streaming
