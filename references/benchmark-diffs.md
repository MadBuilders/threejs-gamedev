# Benchmark Diffing and Run Comparison

## Objetivo
Comparar runs de benchmarks de forma útil y honesta, para detectar regresiones, mejoras reales y tradeoffs sin depender de memoria difusa ni de un único número engañoso.

## Regla principal
**No comparar runs si no son realmente comparables.**
Antes de mirar métricas, validar que contexto y configuración sean equivalentes o que la diferencia esté declarada.

## Qué intenta resolver
- saber si un cambio mejoró o empeoró de verdad
- evitar celebraciones falsas por ruido estadístico o configs distintas
- ver si una mejora de media empeora picos
- poder revisar cambios grandes con criterio reproducible

## Qué no hace por sí solo
- eliminar la variabilidad del hardware o navegador
- sustituir juicio humano sobre calidad visual
- resolver benches mal diseñados

## Primera regla: comparar contexto antes que números
No empezar por “avg frame time”.
Empezar por:
- mismo bench
- misma variante o escenario
- mismo tier
- misma resolución efectiva y `renderScale`
- mismos toggles relevantes
- misma duración de warmup/measure
- misma seed o ruta de cámara cuando aplique

Si eso no coincide, el diff debe marcarlo muy claro.

## Tipos de diff útiles
### 1. Diff de throughput estable
Comparar:
- frame time medio
- p95 o p99
- draw calls medias
- triángulos
- geometries/textures/programs

### 2. Diff de picos/eventos
Comparar:
- peor spike
- pico al activar asset
- pico al cambiar tier
- pico al crear composer o RTT
- pico de spawn/despawn

### 3. Diff de comportamiento adaptativo
Si hay scaler:
- tiempo hasta downgrade
- número de cambios de tier o `renderScale`
- existencia de thrash

## No todos los cambios merecen el mismo peso
Orden sano de lectura:
1. ¿hay diferencia de contexto?
2. ¿cambió el p95/p99?
3. ¿cambió el peor pico relevante?
4. ¿cambió la media?
5. ¿cambió el coste de memoria o draw calls?

Esto evita obsesionarse con una mejora de media minúscula mientras los tirones empeoran.

## Tolerancias
Un diff serio necesita cierta tolerancia para no gritar por ruido.

Ejemplos conceptuales:
- cambios muy pequeños en media, tratarlos como ruido si no se sostienen
- cambios pequeños pero repetidos en p95, prestar más atención
- cambios fuertes en picos máximos, destacarlos aunque la media apenas se mueva

No hace falta fijar números universales en la skill, pero sí dejar claro que:
- no todo delta es significativo
- percentiles y picos suelen importar mucho en juegos

Para convertir esto en thresholds explícitos por proyecto, bench y plataforma, ver `benchmark-thresholds.md`.

## Clasificación útil del resultado
Un diff puede etiquetarse así:
- **mejora clara**
- **regresión clara**
- **mixto / tradeoff**
- **inconcluso**
- **no comparable**

Eso ayuda muchísimo más que soltar una tabla bruta sin interpretación.

## Casos de tradeoff que importan
Ejemplos reales:
- baja la media pero sube el peor pico
- mejora p95 pero aumenta memoria viva
- baja draw calls pero empeora tiempo de build
- mejora tier alto pero rompe tier bajo

Eso no es “éxito” o “fracaso” simple. Es tradeoff, y conviene decirlo así.

## Formato de diff recomendado
### Resumen humano
- qué se comparó
- si es comparable
- veredicto breve
- 3 a 5 cambios más importantes
- observación del tradeoff

### Datos estructurados
Guardar además campos del estilo:
- baseline run id
- candidate run id
- comparable: true/false
- mismatches de config
- delta por métrica
- clasificación final

## Ejemplo de shape útil
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
    "mejora clara en p95",
    "empeora el peor pico al activar bloom",
    "memoria de texturas sube ligeramente"
  ]
}
```

## Qué mismatches invalidan o ensucian mucho la comparación
- distinta resolución efectiva
- distinto tier
- distinto `renderScale`
- distinta duración de la ventana de medida
- distinta seed o ruta si la escena depende mucho de ellas
- distinta variante del bench

Si pasa esto:
- marcar `no comparable` o `comparable con reservas`
- no vender el resultado como definitivo

## Diffs por presupuesto
Muy útil además de deltas absolutos:
- ¿entra aún en el presupuesto?
- ¿sale del presupuesto?
- ¿se aleja peligrosamente del margen?

A veces un cambio empeora un poco pero sigue dentro del objetivo. O mejora media pero sigue fuera del presupuesto crítico.

## Diffs por categoría
Separar al menos:
- rendimiento estable
- picos/eventos
- memoria/recursos
- calidad adaptativa si aplica

Eso hace el diff mucho más legible que una lista plana enorme.

## Integración con benches semiautomatizables
Patrón sano:
- `benchRunner` produce runs estructurados
- `benchDiff` compara dos runs compatibles
- salida final en texto + JSON

No hace falta que todo sea sofisticado desde el día 1. Con consistencia ya paga mucho.

## Integración con revisión humana
El diff no debe sustituir mirar el contexto.

Siempre conviene preguntar:
- ¿el cambio visual merece el coste?
- ¿el empeoramiento aparece solo en una escena rara o en la principal?
- ¿el beneficio es para desktop pero castiga móvil?

## Anti-patrones
- comparar runs con distinta config sin avisar
- celebrar mejoras de media ignorando picos peores
- usar solo FPS en vez de frame time
- no distinguir inconcluso de regresión real
- ocultar mismatches de contexto

## Recomendación fuerte
Crear un pequeño `benchDiff` o equivalente que:
- valide comparabilidad
- calcule deltas por métrica
- agrupe por categorías
- marque mismatches
- emita clasificación final: mejora, regresión, mixto, inconcluso o no comparable

## Pendiente de ampliar
- snapshots visuales coordinadas con el diff
- comparación de series temporales completas
- integración con bots o PR summaries
