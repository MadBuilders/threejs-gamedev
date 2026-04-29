# Persistence and Save

## Goal
Save and load game state, progress, and settings in a Three.js web game without losing data on refresh, without mixing loose formats, and without calling storage from everywhere.

## Main rule
**Gameplay does not touch storage directly.**
Everything persists through a `SaveService` with versions, namespaces, and validation. If tomorrow the project migrates from `localStorage` to `IndexedDB` or to a backend, gameplay does not notice.

## What to persist
- **Settings**: volume per bus, graphics quality, camera sensitivity, language, remapped controls.
- **Progress**: current level, unlocks, collectibles, statistics.
- **Save slot(s)**: continuable state for a specific playthrough.
- Optional **local telemetry** (high scores, times).

Each one with its own namespace and schema.

## What NOT to persist
- derivable state (caches, temporary camera positions).
- pools, textures, geometries.
- debug flags by default.
- anything with references to Three.js objects.

## Web storage: choose deliberately
- **`localStorage`**: enough for settings and small saves (KB). Synchronous, simple. Limit ~5 MB per origin.
- **`IndexedDB`**: for large saves, multiple slots, binary data. Asynchronous. The sensible route once the save grows beyond tens of KB.
- **`sessionStorage`**: useful for things that die when the tab closes, not for real saves.
- **Backend**: only if the game asks for it (cloud saves, leaderboards, accounts). Do not add it reflexively.

Practical rule: settings in `localStorage`, saves in `IndexedDB` unless the save is trivial.

## SaveService shape
Minimum API:
- `loadSettings()` / `saveSettings(settings)`
- `loadProgress()` / `saveProgress(progress)`
- `listSlots()` / `loadSlot(id)` / `saveSlot(id, state)` / `deleteSlot(id)`
- `clearAll()` (with confirmation).

Everything typed. Everything passing through custom serializers.

## Versioning and migrations
**Every payload carries a required `version`.**
On load:
1. read `version`.
2. if it matches the current version, validate and use it.
3. if it is older, apply migrations step by step (v1 → v2 → v3).
4. if it is newer than the known version, reject it and warn (do not try to guess).
5. if there is no `version` or parsing fails, treat it as corrupted.

Migrations as pure functions `(old) => new`. Keep old migrations even if the current version is much higher; a user may come from an old build.

## Validation
Do not trust storage to contain what you wrote:
- runtime schema (manual, Zod, Valibot, according to taste).
- safe fallbacks: if validation fails, load defaults and do not crash.
- log validation errors in development; in production, degrade silently to defaults except for critical cases.

## Serialization
- Plain JSON by default.
- avoid circular references (choose what is saved; do not “dump the whole object”).
- for binary data (saved procedural textures, large snapshots), `IndexedDB` accepts `Blob`/`ArrayBuffer` directly.

Data that should not go to JSON directly:
- `Vector3` → serialize as `{x, y, z}` or an array.
- dates → ISO string.
- `Map`/`Set` → array of pairs or array.

## Auto-save
- clear events trigger saves: level end, checkpoint, pause, critical change.
- throttle: if gameplay emits many events, do not write on every one.
- debounce settings (the user moves a slider; save on release or every N ms).
- never save every frame.

## Data loss and robustness
- write to a temporary slot and rename at the end when possible (in `IndexedDB`, this is handled with transactions).
- keep a previous backup slot; if the current one is corrupted, fall back to the backup.
- do not block the main thread with large saves: `IndexedDB` is already asynchronous.

## Privacy and quotas
- warn if the game requests `navigator.storage.persist()` (persistent mode).
- handle `QuotaExceededError`: offer to clean old saves.
- on mobile, the OS can clear storage; do not assume it always remains.

## Settings and UI
- settings are the first candidate for persistence.
- UI reads and writes through the `SaveService`, never directly to `localStorage`.
- hot-applicable changes (volume, sensitivity) vs changes that require restart (language, graphics API): mark this in the UI.

## Integration with audio and UI
- `AudioService` initializes with loaded settings; if they change, it applies them live.
- control remapping (see `input-controls.md`) persists per profile or slot.
- HUD can show “saving...” but should not block.

## Security and tampering
- in web singleplayer games, encrypting the save is not worth it.
- if the game has online scores, truth must live on the server. The local save is not authority.
- do not store sensitive tokens in `localStorage`.

## Debug
- panel that can dump the current save, import, export, and delete.
- also version the build and commit in the payload to diagnose bugs.

## Anti-patterns
- `localStorage.setItem('player', JSON.stringify(player))` with the entire gameplay object
- saving `Vector3` as-is (the type is lost when parsing)
- not putting `version` on the payload
- migrations that mutate the object and break intermediate saves
- writing every frame
- adding backend cloud saves before validating the local save
- ignoring `QuotaExceededError`
- encrypting singleplayer game saves “for security”

## Strong recommendation
From day 1:
- `SaveService` with namespaces (`settings`, `progress`, `slots/*`).
- `version` on every payload.
- validation + fallback to defaults.
- `localStorage` for settings, `IndexedDB` once the save exceeds a few KB.
- gameplay events trigger saves, with throttle/debounce.

## Related references
- `audio-systems.md`
- `ui-hud.md`
- `input-controls.md`
- `architecture.md`
