# Benchmark Reporting and Semi-Automation

## Objetivo
Convertir escenas de estrés y benchmarks internos en pruebas comparables, con resultados que sirvan para revisar cambios grandes sin depender solo de memoria, intuición o una mirada rápida al FPS.

## Regla principal
**Un bench que no deja rastro comparable se olvida y pierde valor muy rápido.**
Aunque no haya automatización completa, sí debe haber una forma repetible de correrlo y guardar un reporte útil.

## Qué intenta resolver
- comparativas antes/después más honestas
- evitar “creo que iba mejor”
- detectar regresiones visuales o de frame pacing
- tener un lenguaje común para revisar PRs o cambios de arquitectura

## Qué no necesita desde el día 1
- CI perfecta
- granja de dispositivos
- tooling industrial

Con semiautomatización ya se gana mucho si:
- el bench se lanza fácil
- la configuración queda guardada
- las métricas salen en formato comparable

## Nivel mínimo viable
Un bench semiautomatizable debería poder:
1. arrancar una escena concreta
2. fijar una configuración reproducible
3. correr una ventana de medición
4. capturar métricas clave
5. guardar un resultado legible

## Qué capturar siempre
### Contexto del run
- nombre del bench
- fecha
- commit o versión si existe
- dispositivo/navegador si se conoce
- resolución y pixel ratio efectivos
- tier activo
- toggles relevantes: sombras, post, densidad, RTT, etc.

### Métricas mínimas
- frame time medio
- frame time p95 o p99 si puedes
- peor pico relevante
- draw calls
- triángulos
- geometries/textures/programs

### Métricas opcionales según bench
- tiempo de build
- tiempo de carga
- tiempo hasta “asset ready to show smoothly”
- tiempo de spawn/despawn
- latencia del scaler adaptativo en reaccionar

## No medir desde el frame 0 a lo loco
Patrón sano:
- fase de warmup
- fase de medición
- reporte final

Razón:
- al principio hay compilación, carga, cachés y setup
- mezclar eso con throughput estable de gameplay ensucia la lectura

Eso no significa ignorar el arranque, sino medirlo aparte cuando importa.

## Separar throughput de stutter
Conviene tener dos lecturas distintas:

### 1. Régimen estable
- media
- percentiles
- draw calls y estado medio

### 2. Eventos críticos
- pico al activar asset
- pico al cambiar tier
- pico al crear target o composer
- pico al entrar en chunk o encounter

Un bench sano no aplasta todo en un número único.

## Reproducibilidad
Cuanto más controladas estén estas cosas, más valor tiene el reporte:
- seed fija si hay aleatoriedad
- misma ruta de cámara o path pregrabado
- mismo orden de acciones
- misma duración de run
- misma configuración visual

Sin eso, comparar runs se vuelve barro.

## Rutas de cámara y scripts de acción
Muy buena semiautomatización sin volverse loco:
- una ruta de cámara fija
- una secuencia de inputs predefinida
- un timeline simple de eventos

Ejemplos:
- orbitar durante 10s
- activar tier alto en el segundo 5
- spawn de 100 props en el segundo 8
- cambiar skin en el segundo 12

Eso da contexto real a los picos.

## Formato de salida recomendado
No hace falta casarse con uno solo, pero conviene tener al menos:

### Resumen legible
Un markdown o bloque de texto con:
- bench
- config
- métricas clave
- observaciones

### Datos estructurados
JSON o similar con campos estables para comparar runs y, si hace falta, graficar luego.

## Ejemplo de shape útil
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

## Comparación antes/después
Una comparación útil debería poder decir:
- qué cambió
- qué bench se usó
- qué métricas subieron o bajaron
- si el cambio afecta media, percentiles o picos
- si el tradeoff visual compensa

Para la capa concreta de diff entre dos runs con validación de comparabilidad y clasificación final, ver `benchmark-diffs.md`.
Para decidir cuándo esos deltas importan de verdad según proyecto, bench y plataforma, ver `benchmark-thresholds.md`.

## Patrón práctico de semiautomatización
### Paso 1
Crear bench scenes o modos accesibles por query param, debug menu o ruta dedicada.

### Paso 2
Permitir fijar config desde fuera:
- seed
- tier
- variant
- duración
- densidad
- renderScale

### Paso 3
Tener un pequeño recolector de métricas:
- acumular frame times
- capturar percentiles simples
- leer `renderer.info`
- registrar eventos clave

### Paso 4
Emitir resultado:
- a consola
- a overlay debug
- a JSON descargable o copiable

## Qué merece automatización primero
- warmup y ventana de medición
- config estable del run
- ruta de cámara o secuencia de acciones
- resumen final estructurado

No hace falta automatizar primero screenshots, charts o infra complicada.

## Inspiración útil de examples
### `webgl_instancing_performance`
- patrón bueno de variants comparables
- `console.time()` para medir build
- misma escena, distinta estrategia

Ese espíritu merece vivir en benches internos reales.

## Integración con profiling
Los reportes deberían conectar con la doctrina de budgets:
- ¿qué bench rompe el presupuesto?
- ¿qué tier lo arregla?
- ¿qué cambio mejora media pero empeora picos?

## Integración con adaptive quality
Si un bench usa scaler adaptativo, conviene registrar además:
- tiempo hasta primer downgrade
- número de downgrades/upgrades
- si hubo thrash
- si mejoró percentiles o solo media

## Integración con GPU vs CPU heuristics
Si el bench cambia una palanca visual limpia o una lógica limpia, el reporte puede ayudar a clasificar mejor el cuello:
- visual/GPU-ish
- lógico/CPU-ish
- mixed
- stutter/load

## Anti-patrones
- medir a ojo y no guardar nada
- comparar runs con configuración distinta sin decirlo
- mezclar warmup, loading y steady-state en un único número
- capturar solo FPS medio
- no registrar el tier o renderScale activos
- cambiar varias variables fuertes a la vez

## Recomendación fuerte
Crear una capa pequeña de `benchRunner` o equivalente que centralice:
- configuración reproducible
- warmup/measure windows
- captura de métricas
- salida en texto + JSON
- marcas de evento relevantes

## Pendiente de ampliar
- snapshots visuales coordinadas con métricas
- export de series temporales de frame time
- integración con CI o bots de revisión
