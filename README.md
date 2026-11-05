# SCP — Sistema de Conocimiento del Proyecto · MaldoxFilm

Base de conocimiento persistente del proyecto **MaldoxFilm** (migración Vue 3).
Permite reconstruir el contexto técnico y funcional completo en futuras sesiones,
**sin depender de conversaciones anteriores**.

> ⚠️ Esta documentación vive SEPARADA del código fuente.
> Código: `/Users/angelomaldonado/Desktop/maldoxfilm-vue/`
> Original (fuente de verdad histórica): `/Users/angelomaldonado/Desktop/Streamix-local/assets/index.html`

---

## Propósito

MaldoxFilm es una app de streaming (catálogo TMDB + reproductor Real-Debrid)
que se está **migrando desde un monolito vanilla JS** (`index.html` de ~9.600
líneas) **a Vue 3 + TypeScript**, preservando los 20 requisitos originales 1:1
(incluidos sus "puntos ciegos" cosméticos).

---

## Cómo utilizarlo (carga inteligente)

**No leer toda la documentación en cada consulta.** Cargar solo lo necesario.

> 📖 **Índice maestro + tabla de "qué leer por consulta" → `architecture/project-map.md` §0**
> (única fuente; ahí está el catálogo completo de documentos y la búsqueda rápida).

### Al iniciar sesión ("Lee el contexto")
1. `project-memory.md`
2. `current-sprint.md`
3. `architecture/project-map.md`
4. Solo la documentación necesaria según la tarea.

---

## Reglas de mantenimiento

1. **El código es la fuente de verdad.** Si difiere de la doc → actualizar la doc.
   Nunca asumir que la doc está al día; comparar siempre con el estado real.
2. **Al finalizar una tarea**, actualizar:
   - `project-memory.md` (estado, decisiones, riesgos, próximos pasos)
   - `current-sprint.md` (✅/🔄/⏳/⚠)
   - `sessions/YYYY-MM-DD.md` (bitácora del día)
   - Documentación técnica **solo si hubo cambios reales**.
3. `project-memory.md` mantiene el **estado vigente**, NO es bitácora histórica
   (eso va en `sessions/`).
4. Decisiones arquitectónicas importantes → un `adr/ADR-XXX.md` nuevo.

---

## Índice

El **índice maestro** (catálogo de todos los documentos con qué contiene cada uno) vive
en **`architecture/project-map.md` §0** — fuente única, para no duplicar la lista.
