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

## Pendiente de ampliar
- tipos de `shadowMap`
- comparación directional / spot / point en coste real
- política de sombras por preset de calidad
- técnicas híbridas con lightmaps o AO
