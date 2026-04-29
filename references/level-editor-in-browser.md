# Level editor in-browser (Three.js)

Concrete patterns for adding a level editor inside the same app as the game (no external engine, no R3F). Validated in real projects. Complements the **Level editors / authored worlds** block in `SKILL.md`.

## When it applies

- The game has authored content (houses, trees, billboards, waypoints, etc.) and already lives in JSON.
- You want designers (or yourself) to be able to move things without opening an IDE.
- The project has a dev server (Vite, Colyseus, Express, etc.) or you can set up a minimal one.

If the content still fits in 30 lines of constants, wait. The editor is fixed cost; it pays off when the JSON starts hurting to edit by hand.

## Base architecture

### One app, two boots

The editor should **reuse the game bootstrap** (renderer, scene, camera, sky, loaders, PBR). Activate by URL:

```ts
// main.ts
const params = new URLSearchParams(window.location.search);
if (params.get('editor') === '1') {
  await bootLevelEditor(canvas);
  return;
}
// ... normal game boot
```

Advantage: what you see in the editor **is** the game, not an approximation. Materials, shadows, and models load the same way. Visual changes are validated without rebuilding.

### Single source: authored JSON

- `public/levels/<level>.json` (served static) = source of truth.
- Client: `loadLevelDefinitionFromUrl(path)` with fallback to a hardcoded `defaultLevelDefinition()` (useful for offline / corrupt file).
- Schema in `levelDefinition.ts` with `normalizeLevelDefinition(value)`: defensive normalizer that accepts old JSON, fills defaults, accepts new optional fields.
- **Strict separation**: the schema does not import `three`. That way server, scripts, or shared code can read levels without dragging in WebGL.

### The game **must** consume the JSON

Classic bug after adding an editor: the game still calls `createLevel()` with no argument and pulls from the hardcoded default. Always verify:

```ts
const levelDef = await loadLevelDefinitionFromUrl(DEFAULT_LEVEL_PATH).catch(defaultLevelDefinition);
const level = createLevel(levelDef);
```

Without that `await`, the editor saves changes nobody sees.

## Save: from copy/paste to dev endpoint

Healthy progression for the save workflow:

1. **MVP**: "Copy JSON" + "Download" button. It works, but every save is 3 clicks + paste + refresh.
2. **Desirable**: dev endpoint that writes directly to disk.

### Dev-only endpoint on the server

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

Non-negotiable points:

- **Guard `NODE_ENV !== 'production'`** or do not expose the endpoint in prod builds. Writing to disk from a browser POST in production is a security hole.
- **Resolve the path relative to the server binary** (`fileURLToPath(import.meta.url)`), not CWD. If the dev server changes cwd, the save silently corrupts.
- Permissive CORS only in dev (`Access-Control-Allow-Origin: *`).
- **Body limit** (`256 kb` typical). A level JSON should never exceed that; if it does, you have another problem.

### Client: resolve the HTTP base

Do not hardcode `http://localhost:2567`. Reuse the URL params the game already has (e.g. `?mp=`):

```ts
function resolveServerHttpBase(): string {
  const p = new URLSearchParams(window.location.search);
  const mp = p.get('mp');
  if (mp?.startsWith('ws://')) return 'http://' + mp.slice(5).replace(/\/$/, '');
  return 'http://localhost:2567';
}
```

### Draft autosave in localStorage (no modal)

- Every change persists a draft in `localStorage['<app>/editor-draft/<level>']`.
- When opening the editor, if the draft differs from the served JSON, **restore it silently**.
- UI: a chip "• Unsaved changes" / "All changes saved" next to the Save button. No modal asking "do you want to restore?" — it is noise, and it is also what the user always chooses.
- Save clears the draft.
- If someone wants to discard the draft: "Reset to default layout" (in Advanced) + Save.

## Transform gizmo for top-down authoring

### Problem: `TransformControls` has only one mode

`translate`, `rotate`, or `scale` — not all three at once. The "Move / Rotate" toggle in the UI is clumsy: the user always forgets to change it.

### Solution: two `TransformControls` on the same object

```ts
const move = new TransformControls(camera, renderer.domElement);
move.setMode('translate');
move.showX = true; move.showY = false; move.showZ = true;   // top-down: XZ
move.setTranslationSnap(0.5);

const rotate = new TransformControls(camera, renderer.domElement);
rotate.setMode('rotate');
rotate.showX = false; rotate.showY = true; rotate.showZ = false; // yaw only
rotate.size = 1.35;                                              // ring outside the arrows
rotate.setRotationSnap(THREE.MathUtils.degToRad(15));

scene.add(move.getHelper(), rotate.getHelper());
```

When the user selects something:

```ts
move.attach(mesh);
if (supportsRotation(selection)) rotate.attach(mesh); else rotate.detach();
```

### Gate `OrbitControls` with a drag counter

Two gizmos → two `dragging-changed` events. If you use a `boolean`, they desync. Use a counter:

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

### Snap + Shift for free movement

`setTranslationSnap` / `setRotationSnap` enable snapping. **`TransformControls` automatically disables snapping while Shift is held** — you do not need to implement it. Mention it in the hint ("Hold Shift for free motion") because otherwise nobody discovers it.

Reasonable defaults:

- **Rotation: 15°**. Covers 0/45/90/135/180 for free, plus intermediate angles (30/60/75) for props that look bad on pure cardinal directions. 45° is too rigid; 5° is noise.
- **Translation: 0.5 m**. Visible grid in top-down without fighting to align things.

### Rotation buttons as complement, not replacement

The ring with snap works well, but:

- It is not discoverable if you do not see it.
- It is easy to grab an arrow by mistake.
- For exact 90°, click is faster than drag.

Add 4 buttons in the selection card with the same step as snap:

```
[↺ 90°]  [−15°]  [+15°]  [90° ↻]
```

Everything that rotates calls the same `(getYaw, setYaw) => { ... }`; do not hardcode by asset.

Numeric input "type a value in degrees" → removed after feedback: people prefer pressing a button a thousand times over typing. Keep **only** the `<input type="number">` for discrete values that are not angles (e.g. "Tweet index" in billboards).

## Disposal gotcha (the ghost yellow box)

Broken pattern:

```ts
function clearHelpers() {
  disposeObjectTree(helpersGroup);   // frees GPU resources
  scene.add(helpersGroup);           // reattaches the Group
}
```

`disposeObjectTree` calls `geometry.dispose()` and `material.dispose()` on each child, and then runs `helpersGroup.removeFromParent()`. When re-adding it to the scene, **the children are still in `helpersGroup.children`**. They appear as ghost boxes because Three re-uploads buffers on the next draw.

Correct pattern:

```ts
function clearHelpers() {
  for (const child of helpersGroup.children) {
    (child as THREE.Mesh).geometry?.dispose?.();
    const m = (child as THREE.Mesh).material;
    if (Array.isArray(m)) m.forEach(disposeMaterial); else if (m) disposeMaterial(m);
  }
  helpersGroup.clear();  // <-- this is what was missing
}
```

General rule: **freeing GPU resources ≠ removing objects from the graph**. They are two steps.

## Asset catalog (extensibility)

The editor should not know concrete asset names. Use a central catalog `{id, url, label}` by category:

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

Adding a new asset = **3 mechanical steps**:

1. GLB in `public/models/`.
2. Extend the union + `kinds` array in `levelDefinition.ts`.
3. Add entries to URL + label maps.

The editor reads the catalog → the "Type" dropdown fills itself. Do not touch editor code for each new asset.

### Schema backward compatibility

When you add a field like `variant` to houses:

- **Optional** in the type: `variant?: HouseVariantKind`.
- Helper `resolveHouseVariant(slot, index)` that returns `slot.variant ?? HOUSE_VARIANT_KINDS[index % N]` (round-robin fallback).
- Old JSON keeps loading exactly as before. When the user saves, houses they **touched** get an explicit variant; the rest keep the fallback until edited.

## UI: how not to make a miniature Photoshop

Common anti-patterns:

- Mixing "safe operation" buttons (Save, Add, Delete) with the "danger zone" (Reset, Import raw JSON, Rebuild) in the same panel.
- Two selectors horizontally at the top (Mode + Transform). You never touch the second once you understand it.
- Textarea with raw JSON taking half the screen. Useful 1% of the time, noise 99% of the time.
- "Preview building..." / "Preview ready" state that nobody asked for.

Layout that actually works:

```
Level Editor

What to edit: [Houses ▼]
  · short 1-line hint about what drag/ring does

Selection
  · label (House #3)
  · dropdown for variant / kind (if applicable)
  · rotation buttons (if rotatable)

[+ Add house]  [Delete]

─────
● Unsaved changes              [Save to file]
─────

▸ Advanced (collapsed)
  · World boundary radius
  · Auto-generate billboards
  · Download / Import JSON
  · Reset to default layout
```

Concrete tricks:

- **Contextual Add button label**: "+ Add house", "+ Add tree", etc. Changes by mode.
- **Auto-select the newly added item**: it is placed at (0,0); the user drags it into place without an extra click.
- **Save disabled if there are no changes**, green when there are. No confirmation modal needed.
- **Reset asks for `confirm()`**: it is destructive and deserves friction.

## What not to put in the editor (yet)

- **Global undo/redo**: requires command architecture. Start with a `localStorage` draft as a safety net.
- **Multiselect + group operations**: useful in mature editors, unnecessary in the first 20 levels.
- **Day/night preview, world time, simulated physics**: push the user out of the editor to test the game. That is healthy.

## When to add a bake

If you start having LODs, mass instancing, or derived procedural content (ribbon meshes, nav mesh, baked lighting), there comes a point where the client needs a preprocessed artifact, not the direct authored source.

Rule: **the authored source remains the source of truth**. The bake is a build step, versioned or not, but always rebuildable from the source. If the game breaks when the bake is missing, your order is inverted.
