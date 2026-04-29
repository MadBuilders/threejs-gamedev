# Build and Deploy

## Goal
Move from `pnpm dev` to something playable on a public domain without breaking assets, inflating the bundle, or making users drag around stale caches.

## Main rule
**The production game is not the development game.**
Asset compression, cache busting, browser targets, and loading policies are decided before the first deploy, not after the first bug.

## Default stack (see `default-project-stack.md`)
- Vite as the bundler.
- TypeScript.
- `public/` for static files (models, textures, audio).
- `src/` for code.

Vite resolves most healthy decisions by default. Do not change without a reason.

## Browser targets
Define explicitly in `package.json` or Vite config:
- modern desktop (latest 2 major versions of Chrome/Firefox/Safari/Edge).
- modern mobile according to the real target of the game.

Avoid supporting browsers without WebGL2 unless there is a clear requirement. Declare support in the README.

## Code bundle
- no unnecessary giant dependencies. Every addon counts.
- tree-shaking: import from submodules (`three/examples/jsm/loaders/GLTFLoader.js`), not huge barrels.
- code splitting by routes/screens if there is a large menu: gameplay should not pull in the map editor bundle.
- dynamic imports for optional systems (debug panel, benchmarks, level editor).

## Assets: pipeline
- models in **glTF / GLB** with **Draco** or **Meshopt** (see `gltf-pipeline.md`).
- for **reproducible inspection and optimization** of binaries: CLI **gltf-transform** (`@gltf-transform/cli`, see the section with the same name in `gltf-pipeline.md`).
- textures in **KTX2** with **Basis Universal** for texture-heavy games. For small projects, WebP/AVIF is acceptable.
- audio in **ogg/webm-opus** (see `audio-systems.md`).
- sprite/icon atlases for the HUD.

Have a separate asset build step (script), not manual improvisation every time.

## Size and download
Decide strategy before deploy:
- **everything preloaded**: small games. Splash with progress, then gameplay.
- **streaming by level/zone**: more complex, needs a central loader (see `assets.md` and `world-generation.md`).
- **lazy loading optional systems**: debug, editors, stress scenes.

Serve with `Content-Encoding: br` (Brotli) or `gzip`. Verify it in prod; do not assume it.

## Cache busting
- the Vite build adds hashes to bundled asset names.
- assets in `public/` **do not** get hashes by default: that is your responsibility.
  - either add hashes in the asset build pipeline.
  - or version the directory (`/assets/v3/...`) when it changes.
- `index.html` should be served with `no-cache` or `max-age=0` so the user is not trapped on an old version.
- everything else (JS, CSS, hashed assets) can use `immutable, max-age=31536000`.

## Service Worker and offline
- useful for PWA or offline games.
- dangerous if implemented poorly: users can get cached on a broken version.
- if used, plan explicit invalidation when the version changes.
- by default, do not add a SW until there is a clear need.

## HTTPS and embeds
- `AudioContext`, gamepad, fullscreen, and pointer lock require a secure context.
- develop and publish over HTTPS.
- if the game will be embedded in an external iframe, test pointer lock, audio, and fullscreen from the start; breaking very early is better than breaking at launch.

## Hosting
Healthy mix for web games:
- static files + CDN (Netlify, Vercel, Cloudflare Pages, GitHub Pages, S3+CloudFront).
- if there is a backend (multiplayer, leaderboards), separate front and back; do not mix them into a monolith for convenience.
- use a custom domain from the start if the project is “serious”, so URLs do not need to migrate later.

## Environments
- `dev`: HMR, full source maps, debug panel, benches accessible.
- `staging`: production build but debug behind a flag; real browser targets.
- `prod`: debug behind a flag, minimal telemetry if applicable, no placeholder assets.

Environment variables (`import.meta.env.VITE_*`) for flags. Do not leave hardcoded toggles.

## Source maps
- yes, generate them to debug crashes in production.
- do not publish them on the same endpoint as the bundle if you prefer to hide the code: serve them from a private path or load them only when needed.
- minimum: upload them to the error reporting service (if any).

## Crashes and production errors
- `window.onerror` and `onunhandledrejection` hooked to a minimal sink (could be console + localStorage of recent errors, or a service).
- include in reports: build version, commit, WebGL capabilities, user agent.
- do not block the game for non-fatal errors; show a discreet warning.

## WebGL capability check
- detect WebGL2 support at startup.
- show a clear message if the browser/GPU does not allow it; do not leave a black screen.
- detect `OES_texture_float_linear`, specific extensions, and degrade features if they are missing.

## First-load performance
- minimal critical HTML, canvas and splash early.
- defer non-blocking scripts.
- preload critical first-level assets while the renderer initializes.
- reasonable LCP/TTI: a game that takes 20 s without feedback loses users before they play.

## Versioning
- `version` in `package.json`, exposed in the UI (title screen, debug).
- tag every release with a git tag.
- save payload also stores the version to detect incompatibilities (see `persistence-save.md`).

## Minimum CI/CD
- linter + type-check on PRs.
- production build in CI to catch breakage before merge.
- automatic deploy to staging on merges to main.
- manual production deploy, with tag.

You do not need an industrial pipeline. You do need “do not upload to prod by hand from your laptop”.

## Anti-patterns
- `localStorage`/`window` touched directly from environment-dependent code (SSR does not apply here, but testing a prod build without local development does).
- assets in `public/` without a cache-busting strategy.
- aggressively cached `index.html`.
- adding a Service Worker without an invalidation plan.
- publishing source maps on the same public CDN without realizing it.
- not checking WebGL capabilities and leaving a black screen.
- trusting that Brotli is active without verifying it.
- telemetry without consent or clear control.

## Strong recommendation
Before the first public deploy:
- declared browser targets.
- asset pipeline with compression.
- cache busting solved for `public/`.
- uncached `index.html`; everything else hashed + `immutable`.
- minimum error reporting.
- version visible in UI.
- WebGL capability check with a message.

## Related references
- `default-project-stack.md`
- `assets.md`
- `gltf-pipeline.md`
- `audio-systems.md`
- `persistence-save.md`
- `mobile-performance.md`
