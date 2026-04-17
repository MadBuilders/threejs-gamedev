# glTF Pipeline

## Objetivo
Tratar `glTF` y `GLB` como un pipeline de producción real para Three.js, no como un simple `loader.load()` aislado.

## Default principal
Para juegos web nuevos en Three.js:
- usar `glTF` o `GLB` como runtime format principal
- mantener archivos fuente aparte
- centralizar carga con `GLTFLoader`
- usar `LoadingManager` cuando haya varios assets o una pantalla de carga real
- considerar compresión de geometría y texturas solo con pipeline claro

## Regla principal
Separar estas capas:
1. **source assets**
2. **runtime exports**
3. **load orchestration**
4. **instanciación/clonado**
5. **activación visual y cleanup**

## `glTF` vs `GLB`
Regla práctica:
- `GLB` suele ser mejor default de distribución cuando quieres un runtime empaquetado y simple de servir
- `glTF` puede ser útil cuando necesitas inspección fácil o assets externos explícitos

No convertir esto en religión. Lo importante es que el runtime sea consistente y mantenible.

## Source files vs runtime files
Regla fuerte:
- los archivos de editor no son runtime assets
- exportar una versión pensada para juego
- no depender del archivo de Blender, Maya o similar como si fuera el asset final

Mantener claro:
- fuente editable
- export runtime
- variante comprimida si existe

## Checklist de export
Antes de integrar:
- escala correcta
- orientación correcta
- pivots útiles
- nombres estables
- materiales razonables
- jerarquía limpia
- clips de animación con nombre
- texturas con tamaño sensato
- polycount proporcional al uso real

## Orquestación de carga
Cuando haya varios modelos o dependencias:
- usar `LoadingManager`
- exponer progreso al usuario si la espera no es trivial
- no arrancar gameplay serio antes de que los assets críticos estén listos

El manual de `game` es bastante claro aquí: la coordinación de carga y la UI de progreso forman parte del producto, no de un detalle menor.

## Asset registry recomendado
Patrón sano:
- registrar assets por id
- separar metadata de runtime object
- cachear el resultado del load cuando toque
- exponer factories para instancias visuales

Ejemplo conceptual:
- `assets.characters.knight`
- `assets.props.crate`
- `assets.environments.village`

## Clonado e instanciación
No todo asset cargado debe añadirse tal cual a escena.

### Caso 1, una única escena grande
- cargar
- montar
- configurar ownership y lifecycle

### Caso 2, múltiples instancias del mismo asset
- clonar con criterio
- si es personaje skinned, usar `SkeletonUtils.clone()`
- no compartir estado animado por accidente

### Caso 3, muchos objetos repetidos
- evaluar instancing o assets preparados con instancing
- no asumir que clonar cientos de nodos normales es gratis

La example `webgl_loader_gltf_instancing` deja una pista útil: glTF puede convivir con `EXT_mesh_gpu_instancing`, así que parte del coste puede resolverse ya desde el asset pipeline.

## Compresión
La revisión oficial empuja varias ideas distintas:

### Compresión HTTP
Primer win casi gratis:
- servir assets con compresión HTTP correcta
- muchas veces da una mejora enorme sin tocar el contenido del asset

El manual de `game` deja un ejemplo muy claro: varios megas bajan muchísimo solo por compresión del servidor.

### Compresión de geometría y texturas
El example `webgl_loader_gltf_compressed` enseña un camino canónico:
- `KTX2Loader` para texturas comprimidas
- `MeshoptDecoder` para geometría comprimida

Patrón base:
- detectar soporte real del renderer para KTX2
- configurar loaders auxiliares explícitamente
- no meter compresión a ciegas sin validar pipeline y dispositivos objetivo

## Recomendación actual
Con lo revisado en esta ola, la apuesta más sana para proyectos serios sería:
- `GLB` como artefacto runtime principal
- compresión HTTP siempre que puedas
- `KTX2` para texturas si el pipeline lo soporta bien
- `Meshopt` como opción muy seria para geometría
- Draco solo si encaja con tu pipeline y lo has medido, no por reflejo

## Shader warmup y activación visual
La example moderna de `webgl_loader_gltf` deja un detalle muy valioso:
- `renderer.compileAsync()` antes de añadir el modelo puede evitar bloqueos visibles al activar el asset

Esto merece default mental en escenas donde:
- cargas bajo demanda
- cambias de personaje o skin
- entras en zonas nuevas
- presentas modelos grandes al usuario

Para frame pacing, warmup y política general de activación sin tirones, ver `frame-pacing-stutter.md`.

## Entorno y lookdev
Varios examples oficiales mezclan carga de glTF con:
- environment map
- tone mapping
- ajuste de cámara

Esto importa porque un asset puede “verse mal” no por el asset, sino por:
- entorno sin iluminar bien
- tone mapping incoherente
- cámara mal ajustada

No culpar al modelo demasiado pronto.

## Fit de cámara y presentación
Para visores, menús de selección o inspección:
- calcular bounds
- ajustar cámara a selección
- actualizar near/far con criterio

El example oficial lo usa bien como patrón de presentación.

## Lifecycle de modelos cargados
Al cambiar de modelo o reemplazar vistas:
- remover visual de escena
- parar actions o mixers si existen
- limpiar recursos si el asset no va a reutilizarse
- no dejar loads viejos ganar carreras de async

La example oficial moderna también deja una señal útil:
- usar ids o guards de carga para ignorar respuestas antiguas cuando el usuario cambia rápido de asset

## Ownership
Cada modelo cargado debería tener dueño claro:
- viewer
- scene chunk
- enemy factory
- character roster
- skin selector

Sin ownership claro, el caos entra por tres sitios:
- fugas
- dobles cargas
- cleanup roto

## Anti-patrones
- tratar el export del DCC como asset final sin revisión
- cargar glTF desde cualquier archivo sin registry ni coordinación
- mezclar load, gameplay y setup visual en el mismo callback kilométrico
- clonar personajes animados sin `SkeletonUtils.clone()`
- comprimir assets sin pipeline reproducible
- ignorar stutter de compilación de shaders
- no tener estrategia para cancelación lógica o loads obsoletos

## Recomendación fuerte
Si el proyecto pasa de prototipo pequeño, crear explícitamente:
- `assetRegistry`
- `gltfAssetLoader`
- `modelFactory`
- `preloadPhase` o `loadingScreenController`
- `assetLifecycle` o integración con lifecycle general

## Pendiente de ampliar
- variantes por plataforma
- preload por escena o bioma
- streaming de bundles glTF
- validación automática de budgets
- política exacta Draco vs Meshopt según proyecto
