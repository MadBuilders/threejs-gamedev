# Level editor in-browser (Three.js)

Patrones concretos para añadir un editor de niveles dentro de la misma app que el juego (sin engine externo, sin R3F). Validados en proyectos reales. Complementa el bloque **Level editors / authored worlds** del `SKILL.md`.

## Cuándo aplica

- El juego tiene contenido authored (casas, árboles, billboards, waypoints, etc.) y ya vive en un JSON.
- Quieres que los diseñadores (o tú mismo) puedan mover cosas sin abrir un IDE.
- El proyecto tiene servidor de dev (Vite, Colyseus, Express, etc.) o puedes montar uno mínimo.

Si el contenido todavía cabe en 30 líneas de constantes, espera. El editor es coste fijo; paga cuando el JSON empieza a doler al editar a mano.

## Arquitectura base

### Una app, dos boots

El editor debería **reutilizar el bootstrap del juego** (renderer, scene, camera, sky, loaders, PBR). Activación por URL:

```ts
// main.ts
const params = new URLSearchParams(window.location.search);
if (params.get('editor') === '1') {
  await bootLevelEditor(canvas);
  return;
}
// ... boot del juego normal
```

Ventaja: lo que ves en el editor **es** el juego, no una aproximación. Los materiales, sombras y modelos cargan igual. Cambios visuales se validan sin rebuildear.

### Source único: JSON authored

- `public/levels/<level>.json` (servido estático) = source of truth.
- Cliente: `loadLevelDefinitionFromUrl(path)` con fallback a un `defaultLevelDefinition()` hardcodeado (útil para offline / archivo corrupto).
- Schema en `levelDefinition.ts` con `normalizeLevelDefinition(value)`: normalizador defensivo que acepta JSON viejo, rellena defaults, acepta campos opcionales nuevos.
- **Separación estricta**: el schema no importa `three`. Así server, scripts o shared pueden leer niveles sin arrastrar WebGL.

### El juego **tiene que** consumir el JSON

Bug clásico después de montar editor: el juego sigue haciendo `createLevel()` sin argumento y tira del default hardcodeado. Verifica siempre:

```ts
const levelDef = await loadLevelDefinitionFromUrl(DEFAULT_LEVEL_PATH).catch(defaultLevelDefinition);
const level = createLevel(levelDef);
```

Sin ese `await`, el editor guarda cambios que nadie ve.

## Save: de copy/paste a endpoint dev

Progresión sana del workflow de guardado:

1. **MVP**: botón "Copy JSON" + "Download". Funciona, pero cada save es 3 clicks + pegar + refrescar.
2. **Deseable**: endpoint dev que escribe al disco directamente.

### Endpoint dev-only en el server

```ts
// server/src/index.ts
app.options('/dev/*', (req, res) => { /* CORS preflight */ });
app.post('/dev/level', express.json({ limit: '256kb' }), async (req, res) => {
  if (process.env.NODE_ENV === 'production') return res.status(403).end();
  const targetPath = resolve(
    dirname(fileURLToPath(import.meta.url)),
    '../../public/levels/level-01.json',
  );
  await writeFile(targetPath, JSON.stringify(req.body, null, 2));
  res.json({ ok: true });
});
```

Puntos no negociables:

- **Guard `NODE_ENV !== 'production'`** o no expongas el endpoint en builds prod. Escribir al disco desde un POST del navegador en producción es un agujero de seguridad.
- **Resolver la ruta relativa al binario del server** (`fileURLToPath(import.meta.url)`), no al CWD. Si el dev server cambia de cwd el save se corrompe silenciosamente.
- CORS permisivo solo en dev (`Access-Control-Allow-Origin: *`).
- **Límite de body** (`256 kb` típico). Un JSON de nivel nunca debería superar eso; si lo hace, tienes otro problema.

### Cliente: resolver la base HTTP

No hardcodees `http://localhost:2567`. Reusa los params de URL que ya tenga el juego (p. ej. `?mp=`):

```ts
function resolveServerHttpBase(): string {
  const p = new URLSearchParams(window.location.search);
  const mp = p.get('mp');
  if (mp?.startsWith('ws://')) return 'http://' + mp.slice(5).replace(/\/$/, '');
  return 'http://localhost:2567';
}
```

### Draft autosave en localStorage (sin modal)

- Cada cambio persiste un draft en `localStorage['<app>/editor-draft/<level>']`.
- Al abrir el editor, si el draft difiere del JSON servido, **se restaura en silencio**.
- UI: un chip "• Unsaved changes" / "All changes saved" junto al botón Save. No modal preguntando "¿quieres restaurar?" — es ruido y además es lo que el usuario siempre elige.
- Save limpia el draft.
- Si alguien quiere tirar el draft: "Reset to default layout" (en Advanced) + Save.

## Transform gizmo para authoring top-down

### Problema: `TransformControls` solo tiene un modo

`translate`, `rotate` o `scale` — no los tres a la vez. El toggle "Move / Rotate" en la UI es torpe: el usuario siempre olvida cambiarlo.

### Solución: dos `TransformControls` al mismo objeto

```ts
const move = new TransformControls(camera, renderer.domElement);
move.setMode('translate');
move.showX = true; move.showY = false; move.showZ = true;   // top-down: XZ
move.setTranslationSnap(0.5);

const rotate = new TransformControls(camera, renderer.domElement);
rotate.setMode('rotate');
rotate.showX = false; rotate.showY = true; rotate.showZ = false; // solo yaw
rotate.size = 1.35;                                              // ring por fuera de las flechas
rotate.setRotationSnap(THREE.MathUtils.degToRad(15));

scene.add(move.getHelper(), rotate.getHelper());
```

Cuando el usuario selecciona algo:

```ts
move.attach(mesh);
if (supportsRotation(selection)) rotate.attach(mesh); else rotate.detach();
```

### Gating de `OrbitControls` con contador de drags

Dos gizmos → dos eventos `dragging-changed`. Si usas un `boolean` se desincronizan. Usa un contador:

```ts
let dragCount = 0;
const onDrag = (e) => {
  dragCount += e.value ? 1 : -1;
  orbit.enabled = dragCount === 0;
  if (dragCount === 0) { /* commit: rebuild helpers + preview */ }
};
move.addEventListener('dragging-changed', onDrag);
rotate.addEventListener('dragging-changed', onDrag);
```

### Snap + Shift para libre

`setTranslationSnap` / `setRotationSnap` activan el snap. **`TransformControls` deshabilita el snap automáticamente mientras Shift está presionado** — no necesitas implementarlo. Menciónalo en el hint ("Hold Shift for free motion") porque si no nadie lo descubre.

Defaults razonables:

- **Rotación: 15°**. Cubre 0/45/90/135/180 gratis, más ángulos intermedios (30/60/75) para props que quedan feos en cardinal puro. 45° es demasiado rígido; 5° es ruido.
- **Traslación: 0.5 m**. Rejilla visible en top-down sin pelear para alinear cosas.

### Botones de rotación como complemento, no sustituto

El ring con snap funciona bien pero:

- No es descubrible si no lo ves.
- Es fácil agarrar una flecha por error.
- Para 90° exactos, click es más rápido que drag.

Añade 4 botones en la tarjeta de selección con el mismo step que el snap:

```
[↺ 90°]  [−15°]  [+15°]  [90° ↻]
```

Todos los que rotan llaman al mismo `(getYaw, setYaw) => { ... }`, no hardcodees por asset.

Input numérico "type a value in degrees" → quitado tras feedback: la gente prefiere tocar mil veces un botón que teclear. Conservado **solo** el `<input type="number">` para valores discretos que no son ángulos (p.ej. "Tweet index" en billboards).

## Disposal gotcha (la cajita amarilla fantasma)

Patrón roto:

```ts
function clearHelpers() {
  disposeObjectTree(helpersGroup);   // libera GPU resources
  scene.add(helpersGroup);           // re-engancha el Group
}
```

`disposeObjectTree` llama a `geometry.dispose()` y `material.dispose()` en cada hijo, y luego hace `helpersGroup.removeFromParent()`. Al re-añadirlo al scene, **los hijos siguen en `helpersGroup.children`**. Aparecen como cajas fantasma porque Three re-sube buffers al siguiente draw.

Patrón correcto:

```ts
function clearHelpers() {
  for (const child of helpersGroup.children) {
    (child as THREE.Mesh).geometry?.dispose?.();
    const m = (child as THREE.Mesh).material;
    if (Array.isArray(m)) m.forEach(disposeMaterial); else if (m) disposeMaterial(m);
  }
  helpersGroup.clear();  // <-- esto es lo que faltaba
}
```

Regla general: **liberar recursos GPU ≠ sacar objetos del grafo**. Son dos pasos.

## Catálogo de assets (extensibilidad)

El editor no debe saber los nombres concretos de los assets. Un catálogo central `{id, url, label}` por categoría:

```ts
// game/treeModel.ts
export const TREE_VARIANT_URLS: Record<TreeVariantKind, string> = {
  olive: '/models/tree-olive-opt.glb',
  poplar: '/models/tree-poplar-opt.glb',
};
export const TREE_VARIANT_LABELS: Record<TreeVariantKind, string> = {
  olive: 'Olive',
  poplar: 'Poplar',
};
// editor/assetCatalog.ts
export const ASSET_CATALOG = {
  trees: TREE_VARIANT_KINDS.map((id) => ({ id, label: TREE_VARIANT_LABELS[id] })),
  // ... houses, props
};
```

Añadir un asset nuevo = **3 pasos mecánicos**:

1. GLB en `public/models/`.
2. Extender el union + array de `kinds` en `levelDefinition.ts`.
3. Meter entrada en URL + label maps.

El editor lee el catálogo → dropdown de "Type" se rellena solo. No tocar código de editor por asset nuevo.

### Backward-compat del schema

Cuando añades un campo como `variant` a casas:

- **Opcional** en el type: `variant?: HouseVariantKind`.
- Helper `resolveHouseVariant(slot, index)` que devuelve `slot.variant ?? HOUSE_VARIANT_KINDS[index % N]` (round-robin fallback).
- El JSON viejo sigue cargando exactamente como antes. Cuando el usuario guarda, las casas que **tocó** quedan con variant explícita; el resto siguen con fallback hasta que las edite.

## UI: cómo no hacer un Photoshop en miniatura

Anti-patrones frecuentes:

- Mezclar botones de "operación segura" (Save, Add, Delete) con "zona peligrosa" (Reset, Import raw JSON, Rebuild) en el mismo panel.
- Dos selectores en horizontal al top (Mode + Transform). El segundo nunca lo tocas una vez lo entiendes.
- Textarea con el JSON crudo ocupando media pantalla. Útil 1% del tiempo, ruido el 99%.
- Estado "Preview building..." / "Preview ready" sin que nadie lo pida.

Layout que sí funciona:

```
Level Editor

What to edit: [Houses ▼]
  · hint corto de 1 línea sobre qué hace drag/ring

Selection
  · label (House #3)
  · dropdown de variant / kind (si aplica)
  · botones de rotación (si rotable)

[+ Add house]  [Delete]

─────
● Unsaved changes              [Save to file]
─────

▸ Advanced (plegado)
  · World boundary radius
  · Auto-generate billboards
  · Download / Import JSON
  · Reset to default layout
```

Trucos concretos:

- **Label contextual del botón Add**: "+ Add house", "+ Add tree", etc. Cambia según el modo.
- **Auto-select del item recién añadido**: se posiciona en (0,0); el usuario lo arrastra al sitio sin un click extra.
- **Save deshabilitado si no hay cambios**, verde cuando sí. No hace falta mostrar un modal de confirmación.
- **Reset pide `confirm()`**: es destructivo, merece fricción.

## Qué no meter en el editor (todavía)

- **Undo/redo global**: requiere arquitectura de comandos. Empieza con `localStorage` draft como red de seguridad.
- **Multiselect + operaciones en grupo**: útil en editores maduros, sobra en los primeros 20 niveles.
- **Preview de día/noche, tiempo del mundo, física simulada**: empuja al usuario fuera del editor a probar el juego. Eso es lo sano.

## Cuándo añadir un bake

Si empiezas a tener LODs, instancing masivo o contenido procedural derivado (ribbon meshes, nav mesh, baked lighting), llega un punto en que el cliente necesita un artifact preprocesado, no el source authored directo.

Regla: **el source authored sigue siendo la fuente de verdad**. El bake es un paso de build, versionado o no, pero siempre reconstruible desde el source. Si el juego se rompe cuando falta el bake, tienes el orden invertido.
