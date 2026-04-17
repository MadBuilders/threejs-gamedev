# Assets

## Objetivo
Definir un pipeline de assets 3D práctico para juegos en Three.js puro, con foco en estabilidad, claridad y coste razonable para web.

## Default principal
Usar `glTF` o `GLB` como formato principal para assets 3D.

Razones:
- es el formato más natural para Three.js moderno
- transporta mallas, materiales, jerarquía, animaciones y escenas
- reduce conversiones raras a mitad del proyecto
- encaja bien con flujos web y herramientas actuales

## Regla base
Separar el pipeline en cuatro fases:

1. **generación o adquisición**
   - modelado manual
   - librerías externas
   - herramientas generativas como Meshy si encajan
2. **limpieza y validación**
   - escala
   - orientación
   - nombres
   - polycount
   - materiales
3. **compresión y empaquetado**
   - decidir si usar Draco o KTX2 cuando aporte valor real
4. **integración en juego**
   - carga
   - cache
   - instanciación
   - binding con gameplay

## Formatos recomendados

### 3D
- default: `glTF` / `GLB`
- evitar formatos legacy salvo necesidad concreta

### Texturas
- preferir tamaños razonables
- evitar texturas gigantes por defecto
- usar compresión cuando el pipeline lo permita
- mantener convenciones claras de color y maps

### Audio
- fuera de alcance principal de esta referencia por ahora

## Loaders base
Normalmente partir de:
- `LoadingManager`
- `GLTFLoader`
- `TextureLoader`
- loaders adicionales solo si están justificados

Para la parte específica de export, `GLB` vs `glTF`, compresión, `compileAsync`, guards de async y variantes de instanciación, ver `gltf-pipeline.md`.

La revisión del manual refuerza una decisión clara: para juegos nuevos, tratar `glTF` como camino feliz y evitar abrir demasiados frentes con formatos legacy salvo necesidad real.

## Reglas de integración
- No cargar assets desde cualquier parte sin coordinación.
- Centralizar rutas, preload y errores de carga.
- Separar la carga del asset de la lógica de gameplay.
- No asumir que un asset externo viene limpio.
- Crear wrappers o factories cuando un asset tenga configuración repetida.
- Tener una estrategia de liberación de recursos cuando un asset deje de usarse.

## Carga y ciclo de vida
El manual empuja bien una idea importante: cargar es solo la mitad del problema. La otra mitad es saber cuándo mantener, reutilizar o liberar recursos.

Regla práctica:
- si un asset se reutiliza mucho, cachearlo con criterio
- si pertenece a una zona o escena que desaparece, preparar su descarga
- no dejar geometrías, materiales y texturas vivas por accidente durante toda la sesión

La example de instancing/performance deja además otra señal sana: cuando reconstruyas grupos grandes de meshes o cambies de estrategia de representación, limpia explícitamente geometrías y materiales viejos en vez de confiar en que el problema se arreglará solo.

Y el manual de disposal lo deja aún más claro: sacar un mesh de la escena no libera automáticamente geometría, material ni textura. Si un asset ya no se necesita, hay que tener una ruta real de cleanup.

## Checklist de entrada para un asset 3D
Antes de meter un asset en el juego, revisar:

- **escala**: que no venga absurdamente grande o pequeño
- **orientación**: que forward/up encajen con el juego
- **pivot**: que tenga sentido para animación y colocación
- **polycount**: que no sea desproporcionado para su uso real
- **materiales**: que no venga con materiales imposibles o demasiado caros
- **texturas**: tamaños, compresión, nombres y maps correctos
- **jerarquía**: nodos limpios, sin basura innecesaria
- **animaciones**: nombres claros y clips útiles
- **sombras**: revisar si realmente debe cast/receive shadow

## Convenciones recomendadas
- nombres estables para nodos y clips
- carpetas por tipo de asset
- distinguir claramente source files de runtime files
- mantener una lista de assets pesados o problemáticos
- versionar decisiones de pipeline si cambian durante el proyecto

## Meshy y herramientas generativas
Se pueden usar para acelerar, pero con disciplina.

Tratar Meshy como:
- acelerador de prototipos
- opción para concepts o props secundarios
- herramienta que exige revisión manual posterior

No tratar Meshy como garantía de:
- topología buena
- materiales listos para producción
- escalado correcto
- coste razonable para móvil

## Anti-patrones
- meter FBX, OBJ y otros formatos mezclados sin criterio
- usar assets generados sin revisión técnica
- cargar cada asset de forma ad hoc en archivos sueltos
- texturas gigantes porque "se ven mejor"
- resolver problemas de pipeline tarde, cuando ya hay 40 assets dentro
- no tener plan para `dispose()` y limpieza de recursos

## Pendiente de ampliar
- preload y asset registry
- streaming de assets por zonas o chunks
- validación automática de nombres y tamaños
- integración con animaciones y state machines
