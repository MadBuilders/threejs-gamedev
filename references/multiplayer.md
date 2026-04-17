# Multiplayer

## Objetivo
Usar Three.js como capa de presentación dentro de un juego multijugador, sin convertir el scene graph en la fuente de verdad del estado de red.

## Regla principal
**Three.js no resuelve multiplayer.**
Resuelve render, cámaras, materiales, geometría, animación y scene graph. La red, autoridad, reconciliación, snapshots y consistencia pertenecen a otra capa.

## Default recomendado
Para la mayoría de juegos con movimiento libre, acción o colisiones:
- **cliente-servidor**
- **servidor autoritativo** para estado importante
- cliente con representación local fluida
- interpolación para entidades remotas
- predicción solo donde compense de verdad

Para decidir cuándo pasar de este default a rollback, lockstep o hit validation más formal por género, ver `multiplayer-consistency-models.md`.

Si el proyecto es pequeño o por turnos, se puede simplificar. Pero no empezar tratando cada `Object3D` como si fuera una entidad de red completa.

## Separación obligatoria
Separar como mínimo:
1. **network state**
2. **simulation/game state**
3. **presentation state**
4. **scene graph**

Patrón sano:
- la red trae mensajes o snapshots
- el gameplay los traduce a estado del juego
- los systems visuales actualizan `Object3D`, animación, partículas y cámara

Patrón tóxico:
- recibir paquete
- hacer `mesh.position.copy(...)` directamente por toda la app
- usar el scene graph como base del gameplay

## Scene graph no es authority
No usar `Object3D.position`, `quaternion` o jerarquías como verdad principal del mundo compartido.

Mejor:
- mantener entidades con ids estables
- estado serializable separado
- scene nodes como vista derivada

Ejemplo sano de entidad de red:
- `id`
- `type`
- `position`
- `rotation`
- `velocity`
- `animationState`
- `health`
- flags de gameplay relevantes

No enviar:
- referencias a meshes
- materiales
- nodos arbitrarios del scene graph
- objetos gigantes sin esquema claro

## Autoridad
Elegir pronto qué decide el resultado real.

### Servidor autoritativo
Útil para:
- shooters
- acción en tiempo real
- física compartida
- juego competitivo

Ventajas:
- menos trampas fáciles
- colisiones y daño centralizados
- reglas consistentes

Coste:
- más complejidad
- reconciliación
- latencia visible si no se suaviza bien

### Cliente autoritativo o peer-ish
Solo lo recomendaría en:
- prototipos
- cooperativo casual
- juegos lentos o de baja exigencia
- herramientas internas

Si el juego importa de verdad, servidor autoritativo suele ser la apuesta sensata.

## Ticks, frames y snapshots
No mezclar sin pensar:
- **render frames** del navegador
- **simulation ticks**
- **network updates**

Three.js renderiza por frame.
La red normalmente llega a otro ritmo.
La simulación puede ir fija o semi-fija.

Default razonable:
- render desacoplado
- simulación con tick definido
- entidades remotas con buffer corto de snapshots e interpolación

## Interpolación
Para entidades remotas:
- guardar snapshots recientes
- renderizar ligeramente en el pasado
- interpolar entre dos snapshots válidos

Eso suele verse mucho mejor que aplicar cada paquete en cuanto llega.

Patrón más concreto:
- buffer corto de snapshots por entidad
- timestamp de red o tick autoritativo
- presentation time retrasado un poco respecto al “ahora”

Así el cliente puede interpolar de forma estable en vez de pelearse con jitter de llegada.

## Snapshots
Pensar los snapshots como estado compacto y serializable del mundo relevante para ese cliente.

Según el género, puede convenir:
- snapshot global pequeño y simple
- snapshot por interest area
- entidades parciales con campos opcionales

Regla:
- no mandar estado visual interno de Three.js
- mandar estado jugable y derivar la presentación localmente

Buenos campos típicos:
- `tick`
- `entityId`
- `position`
- `rotation`
- `velocity` si ayuda a interpolar o extrapolar
- estados discretos relevantes

## Predicción y reconciliación
Solo meter esto cuando la sensación lo necesite.

Útil para:
- movimiento del jugador local
- inputs muy frecuentes
- acciones donde la latencia se nota demasiado

Regla:
- predecir localmente solo lo imprescindible
- reconciliar con estado autoritativo sin destrozar la presentación
- no extender prediction a todo el juego porque sí

Patrón sano para jugador local:
- guardar inputs con tick local
- simular localmente movimiento o acciones de respuesta inmediata
- cuando llega corrección autoritativa, re-simular desde el último estado confirmado si hace falta
- suavizar la capa visual para que la reconciliación no pegue latigazo feo

Patrón tóxico:
- corregir a base de teleports visibles todo el tiempo
- predecir también entidades remotas sin necesidad clara
- mezclar posición visual corregida con authority lógica sin capas

## Entidades remotas
Cada entidad remota debería tener:
- estado de red serializable
- representación visual local
- lifecycle claro de spawn, update y despawn

Patrón útil:
- `networkEntityMap: id -> entity`
- sistema de spawn visual por tipo
- sistema de update visual desacoplado del transporte de red

En juegos algo más serios, añadir además:
- buffer de snapshots por entidad remota
- interpolador por tipo de entidad
- política de extrapolación corta o freeze si faltan datos

## Animación en multiplayer
No replicar mixers, actions o detalles visuales internos como verdad de red.

Replicar mejor:
- estado alto nivel: `idle`, `run`, `jump`, `attack`, `dead`
- dirección o velocidad relevante
- eventos discretos: `fired`, `hit`, `respawned`

En personajes algo complejos, conviene pensar esto como salida de una animation state machine, no como lista improvisada de clips.

Luego el cliente resuelve:
- clips concretos
- blending
- additive layers
- efectos visuales locales

## Física compartida
Si hay física importante para gameplay:
- decidir dónde corre la física autoritativa
- no confiar en que dos clientes simulen exactamente igual por magia
- usar Three.js para visualizar, no para decidir el resultado real

Three.js puede convivir con la física, pero no la reemplaza.

## Cámara y UX local
La cámara casi siempre es local.
No hace falta replicarla salvo casos concretos.

Regla útil:
- replicar intención y estado del jugador
- no replicar cada detalle de cámara o presentación

La sensación del juego suele depender mucho más de una buena predicción local + cámara local suave que de mandar más datos de cámara por red.

## Interest management
No todos los clientes necesitan todo el mundo todo el tiempo.

Interés puede depender de:
- distancia
- habitación o zona
- línea de visión aproximada
- equipo/facción
- relevancia táctica

Regla:
- la red debería filtrar antes de que Three.js tenga que representar basura irrelevante

Esto pega especialmente fuerte cuando el mundo crece, hay muchos actores o existen RTT/minimaps que podrían tentar a meter demasiado contenido vivo.

## Rollback, lockstep y hit validation
No meter estas palabras como si fueran upgrade automático.

Regla sana:
- snapshots autoritativos + interpolación siguen siendo el default general
- rollback encaja sobre todo en juegos muy sensibles a input
- lockstep encaja mejor en estrategia o simulación por comandos
- hit validation autoritativa importa mucho en acción competitiva aunque no uses rollback total

Si el proyecto ya está en esa zona, leer `multiplayer-consistency-models.md` antes de diseñar el stack de red final.

Defaults rápidos por género:
- shooter o acción competitiva: servidor autoritativo, snapshots frecuentes, predicción local limitada e interest management
- cooperativo PvE: snapshots + interpolación + predicción moderada
- sandbox grande: snapshot parcial por área e interés fuerte
- turnos o baja frecuencia: simplificar y priorizar claridad de estado

## Política de representación
No gastar el mismo presupuesto de red/presentación en todo.

Elegir por entidad:
- jugadores remotos: interpolación cuidada
- proyectiles rápidos: eventos + simulación ligera o autoridad clara
- props secundarios: smoothing barato o updates discretos

Separar además:
- que una entidad no llegue por red
- que exista pero no se renderice
- que exista y se renderice simplificada

Y mantener payloads pequeños y estables:
- ids, números compactos, enums y eventos concretos
- no blobs visuales ni dumps del scene graph

## Errores típicos
- usar meshes como modelo de datos
- acoplar websocket y scene updates directamente
- asumir que el orden de llegada siempre será limpio
- mezclar input local con estado remoto sin capas claras
- replicar demasiada información visual en vez de estado jugable
- intentar resolver cheating solo en cliente

## Recomendación fuerte
Para cualquier juego multijugador serio, crear explícitamente:
- `networkClient`
- `networkStateStore`
- `entityReplicationSystem`
- `presentationSyncSystem`

Three.js debería entrar sobre todo en la última capa.

## Pendiente de ampliar
- multiplayer con física compleja
