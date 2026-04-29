# Audio Systems

## Objective
Provide a healthy audio foundation for Three.js games: coordinated loading, buses, voice pools, and spatial audio, without coupling gameplay to loose `play()` calls.

## Main rule
**Do not call `play()` from anywhere.**
All audio goes through the game's `AudioService`, which knows buses, global volume, concurrent voice limits, and state (muted, window focus, pause).

## What to cover
- short and frequent SFX
- music
- ambiences (long, heavy loops)
- character voices or VO
- spatial audio from world entities

## Base decision: `three/audio` vs direct Web Audio API
- Three.js provides `AudioListener`, `Audio`, `PositionalAudio`, and `AudioLoader`. Enough foundation to start.
- For games with more layers (buses, ducking, dynamic filters, music crossfades), use the Web Audio API directly and expose your own wrapper. `three/audio` falls short for serious mixing.
- On mobile, `AudioContext` may start suspended until the first input: unlock it explicitly on the user's first gesture.

## Minimum buses
- `master`
- `music`
- `sfx`
- `ambience`
- `ui`
- `voice` (if applicable)

Each bus has its own `GainNode` connected to master. User settings change volume by bus, not by sound.

## Asset loading and registration
- Declare the audio set alongside the rest of the game's assets (see `assets.md`).
- Formats: prefer `ogg` or `webm/opus`; `mp3` as fallback.
- Short SFX: decoded in memory (`AudioBuffer`).
- Music and long ambiences: stream with `<audio>` + `MediaElementAudioSourceNode` to avoid inflating memory.
- Register by logical key (`sfx/player-hit`), not by path.

## Voice pool
Hard limits per bus:
- SFX of the same type: collapsing (if N identical sounds are already playing within a short window, replace the oldest or ignore).
- Global maximum concurrent voices per bus.
- Priorities: an important SFX can steal a voice from an irrelevant one.

Without this, a burst of events pins the CPU and saturates the mix.

## Spatial audio
- `PositionalAudio` with `AudioListener` attached to the active camera.
- Define `refDistance`, `maxDistance`, and `rolloffFactor` by source type, not by individual sound.
- In top-down or 2.5D games, use spatial audio carefully: panning can be disorienting if camera and orientation do not match the player.

## Music
- Transitions with crossfade, not abrupt cuts.
- Music by game state, not by loaded level.
- Avoid sophisticated vertical layers until the playable loop is validated (phase 3+).
- Loops with cut points marked during export, not calculated by eye.

## Ducking and priorities
Typical cases:
- VO or dialogue: temporarily lowers music and ambience.
- Critical gameplay hit: small duck on the music bus.

Implement this as short gain transitions on the bus, not by touching individual sounds.

## Pause and window focus
- On focus loss (`visibilitychange`), stop or silence according to policy.
- When entering game pause, silence SFX and ambiences, keep music with a smooth fade down.
- Never silently stop `AudioContext`, or state is lost.

## Mobile
- Unlock on first gesture is mandatory.
- Fewer concurrent voices.
- Prefer shorter audio that is less dense in high frequencies.
- Do not assume the device can decode every format: provide fallback.

## Gameplay hooks
Healthy coupling:
- gameplay emits domain events (`onPlayerHit`, `onStepGrass`, `onPenalty`).
- a subscriber maps events to `AudioService` calls.
- the `AudioService` decides which bus, which pool, which priority.

This lets you silence or remap sounds without touching gameplay.

## Debug
- overlay with active voices by bus
- toggle to solo a bus (music only, SFX only)
- optional audio event log with timestamp

## Anti-patterns
- `new Audio(...).play()` scattered through entities
- sharing one `AudioContext` without unlocking it on mobile
- loading long music as `AudioBuffer` and watching memory explode
- spatial audio without defining `refDistance` and `rolloffFactor`
- music loops with artifacts because the cut point was not exported correctly
- adjusting volume individually instead of by bus
- music changing abruptly when moving from menu to gameplay

## Strong recommendation
Have from the start:
- a single `AudioService` with explicit buses
- API by logical keys
- voice pool and priorities
- standardized `AudioContext` unlock
- user settings persisted by bus (see `persistence-save.md`)

## Related references
- `assets.md`
- `default-content-sourcing.md`
- `mobile-performance.md`
- `persistence-save.md`
- `ui-hud.md`
