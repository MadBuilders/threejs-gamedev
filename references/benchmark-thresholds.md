# Benchmark Thresholds by Project

## Objetivo
Definir umbrales concretos por proyecto para leer diffs de benchmarks con criterio, sin tratar todo delta como importante ni esconder regresiones reales bajo la excusa del ruido.

## Regla principal
**Los thresholds no son universales.**
Deben salir del género, del target de hardware, del presupuesto de frame time y de lo que el proyecto considera aceptable visualmente.

## Qué intenta resolver
- saber cuándo un diff merece atención real
- separar ruido de regresión significativa
- evitar discusiones vagas sobre “parece peor” o “eso es nada”
- convertir `benchmark-diffs.md` en una herramienta accionable de proyecto

## Punto de partida: el presupuesto del juego
Antes de fijar thresholds, dejar claro:
- objetivo principal: 60fps, 30fps o mixto
- hardware objetivo: desktop, móvil, consola-like, gama baja
- escenas críticas: gameplay principal, combate, carga, menú 3D, etc.

Ejemplo conceptual:
- si apuntas a 60fps, el frame budget fuerte es ~16.7ms
- si apuntas a 30fps, ~33.3ms

Los thresholds deben leerse contra ese marco, no en abstracto.

## Thresholds por categoría
### 1. Throughput estable
Aplican a:
- frame time medio
- p95/p99
- draw calls medias
- memoria viva aproximada

Regla sana:
- cambios pequeños de media pueden tolerarse
- cambios sostenidos en p95 suelen importar más
- si p95 se acerca al techo del presupuesto, el threshold debería endurecerse

### 2. Picos y eventos críticos
Aplican a:
- frameTimeMaxMs
- picos al cambiar tier
- picos al activar asset
- picos al cruzar portal o crear RTT
- picos de spawn/despawn

Regla sana:
- aquí los thresholds suelen ser más severos que para la media
- un proyecto puede tolerar 0.5ms de media extra, pero no un pico nuevo de 20ms en gameplay crítico

### 3. Recursos y memoria
Aplican a:
- texturas
- geometrías
- programs
- tamaño de render targets

Regla:
- no mirar solo el delta absoluto
- mirar también si el proyecto ya iba justo en móvil o tiers bajos

## Tipos de threshold útiles
### Threshold de ruido
Debajo de esto, el cambio se considera probablemente no significativo por sí solo.

### Threshold de advertencia
A partir de aquí, el diff merece atención y comentario.

### Threshold de bloqueo
A partir de aquí, el cambio rompe presupuesto o política del proyecto y no debería darse por bueno sin excepción justificada.

## Ejemplo conceptual por proyecto
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

No son números universales. Solo muestran la idea.

## Thresholds por bench, no solo globales
Muy importante.

No tiene sentido usar exactamente los mismos thresholds para:
- un draw-call bench sintético
- un gameplay slice
- un asset activation stress
- un minimap RTT bench

Mejor:
- base global del proyecto
- overrides por bench o familia de benches

## Thresholds por tier o plataforma
También conviene separar:
- desktop alto
- laptop media
- móvil
- tier low/medium/high

Porque un delta tolerable en desktop puede ser bloqueo directo en móvil.

## Cómo leer el diff con thresholds
Patrón útil:
1. validar comparabilidad
2. calcular deltas
3. aplicar thresholds del proyecto y del bench
4. clasificar cada delta: ruido, advertencia o bloqueo
5. emitir veredicto final

## Veredicto final orientativo
- **ok / ruido**
- **vigilar**
- **regresión seria**
- **bloqueante**
- **tradeoff aceptable**

Esto combina mucho mejor con `benchmark-diffs.md` que una lectura plana de números.

## Tradeoffs donde los thresholds ayudan
Ejemplos:
- media sube poco pero p95 cruza threshold de advertencia
- draw calls bajan, pero el pico de activación cruza bloqueo
- memoria sube en desktop aceptable, pero bloquea tier móvil

## Dónde guardar estos thresholds
Lo más sano suele ser:
- config del proyecto
- cerca del `benchRunner` o `benchDiff`
- documentados junto al presupuesto del proyecto

No enterrarlos en notas sueltas que nadie vuelve a mirar.

## Anti-patrones
- usar thresholds inventados sin ligarlos al frame budget
- aplicar los mismos thresholds a todos los benches
- bloquear cambios por ruido minúsculo
- no endurecer thresholds en escenas críticas
- no separar desktop y móvil

## Recomendación fuerte
Crear una política de thresholds con tres capas:
- defaults globales del proyecto
- overrides por bench/familia
- overrides por plataforma o tier

## Pendiente de ampliar
- thresholds por género
- thresholds para memoria más concretos
- integración automática con `benchDiff`
- política de excepciones justificadas
