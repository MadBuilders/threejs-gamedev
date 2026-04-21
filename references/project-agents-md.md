# Project AGENTS.md

## Objetivo
Usar un `AGENTS.md` dentro de cada juego como memoria operativa del proyecto, para registrar decisiones, cambios, convenciones y contexto que no conviene dejar solo en el chat.

## Regla principal
**Si una decisión importa mañana, escríbela hoy.**
No confiar en recordar por magia por qué se eligió una librería, qué carpeta manda o qué bug quedó pendiente.

Regla hermana:
**si el proyecto cambió de fase, refrescar el `AGENTS.md`.**
Un `AGENTS.md` útil no es una foto fija del kickoff para siempre. Si el juego
ya tiene multiplayer, editor interno, audio, nuevo layout de carpetas o una
fase distinta, la memoria operativa tiene que reflejarlo.

## Cuándo crearlo
Crear `AGENTS.md` casi desde el principio cuando el juego pase de idea suelta a proyecto real.

## Qué debería contener
### 1. Contexto del juego
- nombre provisional
- premisa en una frase
- plataforma objetivo
- singleplayer o multiplayer

### 2. Stack elegido
- Three.js puro
- Vite/TS o JS
- Rapier sí/no
- otras librerías importantes

### 3. Convenciones del proyecto
- estructura de carpetas
- naming
- cómo se arrancan builds o tests si existen
- cómo se organizan assets

### 4. Decisiones importantes
Ejemplos:
- por qué no se usa framework UI
- por qué multiplayer se dejó fuera de v0
- qué sistema de cámara manda
- qué física está dentro y qué no

### 4.5 Fase actual
Muy útil dejar explícito:
- fase actual del proyecto
- qué sí entra ahora
- qué queda fuera por decisión

Esto ayuda muchísimo a no intentar hacerlo todo de golpe.

### 5. Log breve de cambios
No hace falta diario kilométrico.
Sí hace falta dejar rastro de:
- cambios importantes
- nuevos sistemas añadidos
- refactors serios
- problemas conocidos
- siguientes pasos claros

### 6. Documentos satélite cuando el proyecto crece
Cuando un subsistema deja de ser “una nota más” y pasa a tener roadmap propio
(por ejemplo multiplayer, economía, herramientas internas o pipeline de
contenido), crear un documento hermano y enlazarlo desde `AGENTS.md`.

Patrón sano:
- `AGENTS.md` mantiene la foto general del proyecto
- `MULTIPLAYER.md` guarda decisiones de red, authority, smoke tests y hosting
- otros documentos similares si aparece otro subsistema igual de denso

## Formato recomendado
```markdown
# AGENTS.md

## Juego
- nombre:
- premisa:
- target:
- modo:

## Stack
- three.js:
- vite:
- typescript:
- physics:
- multiplayer:

## Convenciones
- estructura:
- assets:
- render loop:

## Decisiones activas
- ...

## Cambios recientes
- 2026-04-17: se creó bootstrap inicial y loop base
- 2026-04-18: se añadió controlador de personaje

## Próximos pasos
- ...
```

## Qué no meter
- logs eternos de cada tontería
- secretos o credenciales
- copia de documentación pública
- opiniones vagas sin impacto operativo
- snapshots obsoletos del proyecto que ya no describen el código real

## Relación con memoria y skill
Este `AGENTS.md` no sustituye la skill.
Sirve para aterrizar la skill en un juego concreto.

## Recomendación fuerte
Cuando arranque un juego nuevo:
- crear `AGENTS.md`
- escribir stack y decisiones iniciales
- escribir fase actual y exclusiones explícitas
- ir dejando cambios relevantes y próximos pasos

Y cuando el proyecto madure:
- revisar si la fase actual sigue siendo cierta
- quitar contradicciones con el código real
- enlazar documentos especializados si ya existen

Eso ahorra muchísimo contexto perdido entre sesiones.

## Referencias asociadas
- `game-kickoff-planning.md`
- `default-project-stack.md`
- `default-content-sourcing.md`
