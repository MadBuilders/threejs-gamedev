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

## Animación: cuidado con tracks de **scale**

En rigs exportados desde herramientas de IA o con retarget ruidoso, un clip puede llevar **keyframes de scale** en huesos raíz o torso. En reproducción eso se traduce en “inflado” o clipping durante el walk.

Opciones:
- arreglar en DCC / re-export limpio;
- en runtime, **eliminar tracks de scale** del `AnimationClip` al cargar (quedan posición y rotación), si el modelo ya tiene escala correcta en bind pose.

Relacionado: `animation-systems.md` y bounding boxes en skinned meshes (`Box3.setFromObject(..., true)`).

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

#### Migración de `clone()` a `InstancedMesh` por leaf
Patrón recurrente: una función `loadXxxModel()` devuelve un wrapper que expone `instance()` implementado como `wrapper.clone(true)`. Sirve hasta que el nivel escala a decenas o cientos de copias del mismo GLB: cada instancia aporta sus propios `Object3D` (transform, matrix world update) y una draw call por leaf mesh del wrapper.

Plantilla reutilizable para extender el mismo factory con instancing de verdad, sin obligar a cambiar el resto del juego:

```ts
// 1) Al cargar el GLB, extraer los leaf meshes aplanados con su matriz
//    local respecto al wrapper (la que ya aplica escala, recentrado, etc.).
const leafMeshes: { geometry: THREE.BufferGeometry; material: THREE.Material; baseMatrix: THREE.Matrix4 }[] = [];
wrapper.updateMatrixWorld(true);
wrapper.traverse((obj) => {
  const mesh = obj as THREE.Mesh;
  if (!mesh.isMesh) return;
  leafMeshes.push({
    geometry: mesh.geometry,
    material: mesh.material as THREE.Material,
    baseMatrix: mesh.matrixWorld.clone(),
  });
});

// 2) Nuevo método: una InstancedMesh por leaf, N matrices.
function createInstancedMeshes(placements: ReadonlyArray<{ x: number; z: number; yaw: number }>) {
  const group = new THREE.Group();
  const tmp = new THREE.Matrix4();
  const placement = new THREE.Matrix4();

  for (const leaf of leafMeshes) {
    const im = new THREE.InstancedMesh(leaf.geometry, leaf.material, placements.length);
    im.castShadow = castShadow;
    im.receiveShadow = true;
    // Con instancias esparcidas por todo el mundo, el frustum culling
    // per-instance puede esconder instancias reales; suele merecer la pena
    // dejarlo a cargo del bounding sphere global.
    im.frustumCulled = false;

    for (let i = 0; i < placements.length; i++) {
      const p = placements[i]!;
      placement.makeRotationY(p.yaw).setPosition(p.x, 0, p.z);
      tmp.multiplyMatrices(placement, leaf.baseMatrix);
      im.setMatrixAt(i, tmp);
    }
    im.instanceMatrix.needsUpdate = true;
    group.add(im);
  }
  return group;
}
```

Puntos a cuidar:
- **Agrupar placements por variante** antes de llamar al instanced factory: si tu juego mezcla "árbol A" y "árbol B", son dos `InstancedMesh` distintos (no los fuerces a uno mismo aunque compartan escala).
- **Conservar `instance()` para casos raros**: obstáculos interactivos, props que necesitan animación propia o swap de material; esos siguen con `clone(true)`. El instanced path es el default para "decoración densa".
- **Los obstáculos de colisión que apuntaban a la instancia individual ahora deben apuntar al grupo instanced completo** (si los necesitas). Verifica dónde se usa ese `visual` (en muchos juegos solo para debug o para hacer un swap puntual); si el único lifecycle es "vive desde boot hasta fin del nivel", compartir la referencia es seguro.
- **Draw calls**: pasas de `N × leafCount` a `leafCount` (una por submesh del GLB), no "a 1" salvo que el GLB tenga un único mesh.
- **Sombras**: `InstancedMesh` casta sombra por instancia igual que un mesh normal; el shadow pass también se beneficia del colapso de draw calls.
- Si los GLBs llevan `EXT_mesh_gpu_instancing` de origen, este paso en runtime sobra: resuelve ya el pipeline.

## AI-generated GLBs (Meshy y similares)

Los generadores de assets 3D por IA (Meshy, Rodin, Luma, etc.) aceleran mucho el lookdev pero **vienen con defaults silenciosos que hay que corregir siempre** antes de meter el GLB en el juego. Tratarlos como "source editable con lookdev bonito", no como runtime assets.

Checklist obligatorio tras descargar cualquier GLB de IA:

- **`doubleSided: true` en todos los materiales**, incluso en opacos (troncos, paredes, vasijas). Para geometría cerrada es puro coste de fragment shader ~2× sin ganancia visual. Hay que forzar `FrontSide` — ver más abajo.
- **Texturas a 2048×2048 por defecto**, cada slot PBR (baseColor + normal + metallicRoughness + emissive). Un solo árbol puede pedir ~90 MB de VRAM. Bajar a 512 o 1024 según uso real.
- **Sin compresión de geometría ni texturas**. `meshopt` + `webp`/KTX2 como paso automático del pipeline.
- **Polycount descontrolado**. Un mesh "de presentación" puede venir a 0.5M–3M tris. Para realtime: foliage lejano 10K, prop de escenario ~10–30K, edificio ~20–30K. Subir solo si la silueta lo pide de verdad.
- **Skinned meshes: no pasar por AI-remesh**. Las herramientas de IA rompen weights, skeleton y animaciones. Si hay que reducir un personaje animado, el camino es Blender Decimate preservando grupos de vértices, y reverificar cada clip.

### Pipeline post-download canónico

```bash
# Bajar resolución de texturas al tamaño real de uso
pnpm dlx @gltf-transform/cli@latest resize in.glb tmp.glb --width 512 --height 512

# Re-encode a WebP (normal maps y demás). KTX2 si el runtime lo soporta.
pnpm dlx @gltf-transform/cli@latest webp tmp.glb tmp2.glb

# Compresión de geometría
pnpm dlx @gltf-transform/cli@latest meshopt tmp2.glb out.glb
```

Luego `inspect` para confirmar que el resultado es el esperado (tris, `doubleSided`, resolución de texturas, extensiones aplicadas).

### Forzar `FrontSide` en el loader, no en el GLB

Arreglar `doubleSided` con `gltf-transform` o re-export también funciona, pero **es más robusto hacerlo en el loader del juego**:

```ts
gltf.scene.traverse((obj) => {
  const m = obj as THREE.Mesh;
  if (!m.isMesh) return;
  const mats = Array.isArray(m.material) ? m.material : [m.material];
  for (const mat of mats) {
    if (mat) mat.side = THREE.FrontSide;
  }
});
```

Ventaja: cubre cualquier re-export futuro (otra generación, otra herramienta, otro artista) sin depender de que alguien se acuerde de pasar el pipeline. Un solo punto de control en el loader por tipo de asset (árboles, props, casas) evita que la "optimización" se quede anclada al GLB concreto.

### Variant scale como dato authored, no baked in GLB

Si un asset se va a regenerar (textura nueva, mesh mejorada, variante estilística), **no bakear el tamaño final en el GLB** — mantenerlo como constante en código/JSON:

```ts
export const TREE_VARIANT_SCATTER_SCALE: Record<TreeVariantKind, number> = {
  olive: 1.0,
  poplar: 1.3,
  'poplar-alt': 1.8,
};
```

Así puedes re-remeshar el asset sin retunear placements en toda la escena. Es la misma doctrina de **"source editable → artifact"** aplicada a dimensiones: el GLB es el artifact, las decisiones de "qué tan grande va en el mundo" son source humano.

Aplicable a árboles, props, edificios, cualquier cosa que puedas querer refrescar la malla en el futuro.

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

## gltf-transform (CLI)

[gltf-transform](https://gltf-transform.dev/) es la herramienta de referencia para **inspeccionar, limpiar y optimizar** glTF/GLB en pipeline reproducible. No sustituye a Blender/Substance para authoring, pero sí a **export ad hoc** y a “bajar megas” antes de subir a `public/`.

**Paquete:** `@gltf-transform/cli` (el binario suele invocarse como `gltf-transform` vía `npx` o `pnpm dlx`).

**Cuándo usarla**
- Antes de integrar un GLB enorme: entender qué pesa (geometría vs texturas vs extensiones).
- Antes de producción: deduplicar accessors, simplificar materiales, comprimir geometría (p. ej. Meshopt) o texturas según el proyecto.
- En CI: validar que un export no ha crecido más de un umbral (combinar con `benchmarking.md` si aplica).

**Comandos típicos (ejemplos)**

```bash
# Estructura, tamaños, meshes, animaciones, texturas
npx @gltf-transform/cli inspect modelo.glb

# Optimización general (revisar flags en la doc del paquete; evolucionan entre versiones)
npx @gltf-transform/cli optimize entrada.glb salida.glb
```

**Reglas sanas**
- Fijar **versión mayor** del CLI en el proyecto (script en `package.json` o documentado en README) para que `optimize` sea reproducible entre máquinas.
- Tras optimizar, **probar en el juego real** (Three.js + extensiones que uses: Meshopt decoder, KTX2, etc.).
- No tratar la compresión como magia: si el asset sigue gigante, el cuello a menudo son **texturas 4K** u opciones de export del DCC.

**Anti-patrones**
- Optimizar una sola vez “a mano” sin script ni versión fijada y olvidar cómo se regeneró el artefacto.
- Asumir que `optimize` siempre baja calidad visual: depende de flags y del contenido.

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
