# Lights and Shadows

## Objetivo
Tomar decisiones de iluminación y sombras en Three.js con mentalidad de juego web, no de demo aislada.

## Regla principal
Las sombras son caras. Muy caras si se dejan crecer sin control.

El manual lo deja clarísimo: cada luz que genera sombras obliga a renderizar la escena extra desde el punto de vista de esa luz. Con varias luces, el coste se multiplica muy rápido.

## Default recomendado
- empezar con iluminación simple
- usar pocas luces importantes
- si hace falta sombra dinámica, preferir **una directional light principal** antes que varias luces con sombra
- tratar las sombras como presupuesto limitado, no como valor por defecto universal

## Estrategia de sombras por niveles

### Nivel 0, sin sombras dinámicas
Usar:
- iluminación simple
- AO o light hints en assets
- contraste de materiales
- composición visual del escenario

### Nivel 1, fake shadows
Muy recomendables en muchos juegos estilizados o móviles.

Patrón clásico:
- plano o decal simple
- textura de sombra suave en escala de grises
- `MeshBasicMaterial`
- `transparent: true`
- `depthWrite: false`
- colocar ligeramente por encima del suelo para evitar z-fighting

Esto es baratísimo y muchas veces da el pego de sobra.

### Nivel 2, shadow maps reales
Usarlas cuando de verdad mejoran la lectura o la fantasía del juego.

Reglas:
- activar `renderer.shadowMap.enabled` solo cuando toca
- marcar `castShadow` y `receiveShadow` con criterio objeto por objeto
- ajustar la shadow camera de la luz, no dejarla absurda por defecto
- medir el coste en escenas reales

## Luces
No meter luces porque sí.

Preguntas útiles:
- ¿qué aporta esta luz a la lectura del juego?
- ¿podemos conseguir casi lo mismo con una luz menos?
- ¿el estilo visual necesita realismo o claridad?

## Shadow camera
El manual insiste en algo importante: si faltan sombras o salen raras, muchas veces el problema no es "Three.js" sino la región que cubre la shadow camera.

Regla práctica:
- visualizarla con `CameraHelper`
- ajustar top, bottom, left, right, near y far al área útil real
- evitar cajas gigantes si la acción ocurre en una zona pequeña

### Shadow camera que sigue al foco
Para un juego con una **zona jugable grande** y sombras que importan cerca del jugador, la tentación es agrandar el frustum de la shadow camera hasta cubrir el mapa entero. Eso sale caro dos veces: baja la resolución efectiva (cada texel del shadow map cubre más mundo) y obliga a subir `mapSize` para compensar.

Patrón más barato: **mantener un frustum ajustado pequeño y moverlo con el jugador** (o con el foco de cámara).

Receta:
- En boot, guarda el offset original del sol como `SUN_OFFSET = sun.position.clone()`.
- `scene.add(sun.target)`: la `DirectionalLight` apunta desde `position` hacia `target.position`, así que necesitas el target en escena para que la matriz se actualice.
- Cada frame, tras mover al jugador:
  ```ts
  sun.position.set(player.x + SUN_OFFSET.x, SUN_OFFSET.y, player.z + SUN_OFFSET.z);
  sun.target.position.set(player.x, 0, player.z);
  sun.target.updateMatrixWorld(true);
  ```
- Con el frustum parked sobre el jugador, puedes usar half-extents del orden de "lo que la cámara tercera persona ve cerca" (por ejemplo ~20-25 m) y `mapSize` más bajo (`512²` o `768²`) sin perder nitidez.

Tradeoffs:
- la dirección del sol sigue constante (solo se desplaza, no rota), así que la iluminación queda estable aunque el jugador avance.
- sombras lejos del jugador desaparecen; para un juego single-player tercera persona suele ser lo que quieres. Si hay gameplay dependiente de sombras lejanas (sigilo, puzzles), este patrón no sirve.
- combina bien con `PCFShadowMap` (más barato) en vez de `PCFSoftShadowMap`; con el frustum ajustado no suele hacer falta el blur extra.
- si bajas mucho `mapSize`, sube un poco `shadow.bias` negativo (`-0.0005` es un punto de partida sano) para evitar acne en el suelo.

### `receiveShadow` también cuesta por fragmento

Es fácil pensar solo en `castShadow` como la palanca cara (es la que dispara el shadow pass extra). Pero `receiveShadow = true` añade **una muestra de shadow map por cada fragmento del mesh** en el render principal — es un PCF lookup que, en fragmento denso, se nota.

Dónde importa:
- **foliage denso**: copas de árboles, hierba, hojas. Decenas de miles de fragmentos visibles por frame, y el canopy está *por encima* del jugador → recibir sombra ahí no aporta (casi) nada visualmente.
- **superficies que nunca reciben una sombra realista**: cielos, fondos, caras internas de edificios a los que la luz principal no llega.

Patrón sano:
- `castShadow` por gameplay (proyectar sombra del jugador, enemigos, props interactivos).
- `receiveShadow` solo si **realmente** va a caer una sombra de algo encima. No por defecto universal.
- Para árboles scattered: `receiveShadow = false` es casi siempre la respuesta correcta. El suelo sí lo necesita; la copa no.

Exponer el flag como opción en los factories de carga (`loadTreeModel({ receiveShadow: false })`) permite que árboles y props compartan loader pero diverjan en política de sombras: los props que se apoyan en el suelo sí quieren que la sombra del jugador caiga encima, los árboles esparcidos no.

## Helpers y debug
Útiles cuando se trabaja con sombras:
- `CameraHelper` sobre la shadow camera
- herramientas para ver shadow map si el caso se complica
- toggles de HUD o debug para comparar con y sin sombras

## Anti-patrones
- varias luces con sombra por defecto
- `castShadow` y `receiveShadow` activado en todo
- usar shadow maps complejos en móvil sin medir
- no ajustar la shadow camera
- perseguir sombras perfectas cuando el estilo del juego no lo necesita

## Recomendación fuerte
Para muchos juegos web, la combinación ganadora es:
- una luz principal bien elegida
- fake shadows para secundarios
- o directamente sombras muy selectivas solo donde ayudan de verdad

## IBL con HDRI (image-based lighting)
Técnica desproporcionadamente útil para escenas PBR: cargar un HDRI equirectangular, pasarlo por `PMREMGenerator` y asignarlo tanto a `scene.background` como a `scene.environment`. Con una sola llamada obtienes:
- **skybox** detrás de todo (resuelve "el cielo se ve feo"),
- **reflexión + diffuse ambient** coherentes para todos los materiales PBR, sin añadir luces extra.

Receta mínima:
```ts
// Three r168+ sustituyó RGBELoader por HDRLoader. API equivalente.
const hdr = await new HDRLoader().loadAsync(url);
const pmrem = new THREE.PMREMGenerator(renderer);
const envRT = pmrem.fromEquirectangular(hdr);
hdr.dispose();
pmrem.dispose();
scene.background = envRT.texture;
scene.environment = envRT.texture;
```

Cuándo compensa:
- cualquier escena outdoor con materiales PBR
- quieres que personajes, props, metales se iluminen "bien" sin pelearte con 3-4 luces puntuales
- la cámara ve cielo real y no quieres un fondo pintado

Gotchas:
- si ya tenías `HemisphereLight` o luces ambientales fuertes, **bájalas cuando metas env map**, o todo queda doblemente iluminado y se lava el contraste.
- la `DirectionalLight` que representa el sol sigue siendo necesaria si quieres sombras (el env map por sí solo no las proyecta).
- dirección del sol del HDRI ≈ dirección de tu `DirectionalLight`, o cantará.
- Poly Haven (CC0) es el default razonable; 1K basta casi siempre para fondo, 2K si el cielo está muy en pantalla.
- descomprimir un HDR a PMREM cuesta; hacerlo una vez en boot, no por frame.
- disposer el `RenderTarget` de PMREM si la escena se destruye (lo que devuelve `fromEquirectangular` es un `WebGLRenderTarget`).

Anti-patrón: `new RGBELoader().load(...)` y asignarlo directamente a `scene.environment` sin PMREM. Aparenta funcionar pero las reflexiones salen con artefactos y la iluminación ambiental está mal prefiltrada.

`backgroundIntensity` (Three r155+) permite bajar el brillo del cielo visible sin tocar la intensidad del IBL sobre los materiales, útil cuando el HDRI es demasiado luminoso como fondo pero sí sirve como IBL.

## Pendiente de ampliar
- tipos de `shadowMap`
- comparación directional / spot / point en coste real
- política de sombras por preset de calidad
- técnicas híbridas con lightmaps o AO
