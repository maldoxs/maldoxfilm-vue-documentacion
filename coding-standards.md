# Coding Standards — MaldoxFilm

> Convenciones, patrones obligatorios y reglas de desarrollo.

---

## 1. Stack y versiones

- **Vue 3.5** (Composition API, `<script setup lang="ts">`).
- **TypeScript ~6.0** (`vue-tsc` para type-check; el build hace `vue-tsc -b && vite build`).
- **Pinia 3** (stores).
- **Vue Router 4**.
- **Vite 8** (dev/build).
- **Vitest 4** (unit tests, entorno `jsdom`).
- **Netlify Functions** (Node) + `@netlify/blobs`.

---

## 2. Regla maestra: fidelidad 1:1 con el original

El monolito `Streamix-local/assets/index.html` es la **fuente de verdad histórica**.

- Se preservan comportamientos EXACTOS, **incluidos quirks/puntos ciegos cosméticos**.
  Ejemplo: tras inyectar subtítulos, el status siempre muestra el literal
  `✅ ES | offset: 0s` aunque el offset real sea otro (índex.html línea ~5189).
- Cada función/bloque migrado lleva un comentario `// preserva ... (índex.html línea ~XXXX)`
  apuntando a su origen. **Mantener esas referencias.**
- Si dudas de un comportamiento, **leer el original** antes de "corregir". Muchos
  detalles "raros" son intencionales.

---

## 3. Arquitectura por capas (separación pura vs efectos)

Patrón dominante en el proyecto:

```
services/   → lógica PURA (sin DOM, sin red directa salvo "clients") → 100% testeada con Vitest
composables/→ ORQUESTACIÓN (tocan DOM, red, timers, MediaSource) → no testeable con Vitest puro
stores/     → estado reactivo (Pinia) + persistencia (localStorage)
components/  → UI declarativa (props in, eventos out)
views/      → páginas (rutas), arman composables + componentes
```

- **La lógica de "qué hacer" se extrae a funciones puras** en `services/` (testeadas);
  el composable solo "aplica" esos resultados sobre el DOM/red.
- Ejemplo: `streamSelector.ts` (puro: scoring/matching de streams) vs
  `usePlayer.ts` (efectos: `<video>`, Shaka, hls.js, timers).

---

## 4. Convenciones de nomenclatura

| Tipo | Convención | Ejemplo |
|---|---|---|
| Componente `.vue` | PascalCase | `MovieCard.vue`, `HeroBanner.vue` |
| Composable | `useXxx.ts` (camelCase) | `usePlayer.ts`, `useSubtitles.ts` |
| Servicio | sustantivo camelCase | `streamSelector.ts`, `tmdb.ts` |
| Store Pinia | `useXxxStore` en `stores/xxx.ts` | `usePlayerStore` |
| Evento emitido | kebab-case | `@select`, `@fallback-to-next-source` |
| Ruta | kebab-case | `/ver-todo`, `/mi-lista` |

---

## 5. Patrones obligatorios

### Inyección de dependencias de red
Los clientes de red (TMDB/Torrentio/RD/RdStreamResolver) se arman **una vez** en
`services/bootstrap.ts` (`createAppServices`) y se proveen vía
`app.provide(APP_SERVICES_KEY, ...)` en `main.ts`. Se consumen con
`useAppServices()`. **No instanciar clientes sueltos en componentes.**

### RD_TOKEN nunca client-side
Regla de desarrollo: **prohibido** hardcodear el token, usar `.env` con prefijo `VITE_`
o committearlo. Se inyecta vía `bootstrap.ts` (`fetchRdToken` → function `rd-token`).
Política y justificación completas → `architecture/security.md` y `adr/ADR-002.md`.

### Cancelación de async stale (generaciones)
El reproductor usa un contador de "generación" (`playerStore.generation`) que se
incrementa en cada carga. Cada callback async chequea `playerStore.isStale(myGen)`
antes de actuar (equivalente al `if(_playerGen !== myGen) return;` del original).

### Estado reactivo vs punteros
Instancias de terceros (Shaka, hls.js) se guardan en **variables de módulo**, no en
`ref` (no son estado de UI). El estado de UI sí va en `ref`/store.

### Persistencia
`localStorage` vía `services/safeStorage.ts` (wrapper seguro). Keys:
`sx_mylist`, `sx_prog_v2`.

---

## 6. Estilos (CSS)

- **Scoped styles** por componente (`<style scoped>`).
- ⚠️ **FOOTGUN: `:global(html.tv-mode) .x:focus` / `:hover` en scoped CSS compila MAL** — el
  plugin de Vue/postcss descarta la parte después de `:global()` y deja el selector pelado
  (`html.tv-mode`), aplicando el estilo a TODO el `<html>` (causó la "línea azul" en TV,
  commit d74e3be). Los estilos de **foco/hover dependientes de `html.tv-mode` van en
  `src/style.css` global** (sin `:global()`), con `:focus-visible`. `:global(html.tv-mode) .x { ... }`
  SIN pseudo-clase sí funciona (p.ej. tamaños). Diagnóstico: inspeccionar el CSSOM en vivo.
- Variables CSS globales en `src/style.css` (`--accent: #3d5afe`, `--bg: #0e0e0e`,
  `--red: #e05c3a`, `--surface*`, `--text*`, `--radius`, `--trans`, etc.).
- Fuentes: **Oswald** (logo/hero/TOP/títulos) + **Roboto**, cargadas en `index.html`
  vía Google Fonts. Body usa Inter/Roboto.
- **Padding horizontal estándar de secciones/carruseles: `52px`** (alinear cabeceras,
  géneros y rails a ese valor).
- Carruseles: `overflow-y: hidden` obligatorio (si no, `overflow-x:auto` activa
  `overflow-y:auto` implícito y atrapa el scroll vertical de la página).
- **Memoria/perf — `content-visibility: auto`**: ⚠️ **REMOVIDO del código (a pedido del usuario:
  no le gustaba el efecto de imágenes "apareciendo" al scrollear).** Ya NO está en
  Carousel/ContinueWatching/TopNumbered (`grep content-visibility src/` → vacío). Documentado aquí
  para que no se reintroduzca pensando que falta. (corregido 2026-06-13: el doc estaba desactualizado).

### Lecciones TV / webOS (smart-TV LG) — sesión 2026-06-13

- **NUNCA scrollbar horizontal visible.** El original tenía `body{overflow-x:hidden}`; restaurarlo.
  Un `width:100vw` (incluye el gutter del scrollbar vertical) desborda ~8px y aparece un scrollbar
  horizontal al pie cuyo thumb, al hover, se ponía azul (`::-webkit-scrollbar-thumb:hover`). En
  `src/style.css`: `body{overflow-x:hidden}` + `::-webkit-scrollbar:horizontal{display:none}` + hover gris.
- **Fullscreen API vs `position:fixed` en webOS.** El navegador de la TV solo oculta su barra (URL/
  controles) de forma fiable con el Fullscreen API. PERO un `position:fixed` DENTRO de un elemento
  en `:fullscreen` deja de pegarse al scrollear (bug webkit). ⇒ Navs/elementos que deban sobrevivir
  al fullscreen van con **`position: sticky`**, no `fixed`. Patrón usado: `closePlayer` pide
  `requestFullscreen` (gesto del usuario) y `NavTV` es `sticky`.
- **`useFullscreen` no toca el Fullscreen API nativo en TV** durante la reproducción (el overlay CSS
  del player ya cubre la pantalla). Tocarlo togglea la barra del navegador.
- **Skeletons grises en TV:** en `html.tv-mode` el `.card-skeleton` es `background:transparent`
  (reserva el espacio, sin gris ni shimmer). El usuario no quiere ver "tarjetas cargando".
- **Cache de listas (anti-skeleton + perf):** `useTmdbList` cachea con stale-while-revalidate
  (memoria de sesión + `safeStorage`/localStorage TTL 6h). Listas ya vistas se pintan al instante.
- **Pendiente conocido:** la barra del navegador webOS aún "falta afinar"; solución definitiva =
  empaquetar como **PWA / webOS .ipk** (sin chrome de navegador).

---

## 7. Testing

- `npx vitest run` debe quedar **verde** antes de dar por terminada una tarea.
- `npx vue-tsc --noEmit` debe dar **EXIT=0**.
- Los tests cubren las funciones PURAS de `services/` (scoring, matching, catálogo,
  subtítulos, progreso, navegación, etc.). Los composables con efectos no se testean
  con Vitest (requieren browser real).
- Al añadir un campo a una interfaz compartida (p.ej. `SelectedStream`), actualizar
  los `toEqual()` de los tests afectados.

---

## 8. Flujo de trabajo / git / deploy

- **Probar SIEMPRE en local primero** con `netlify dev` (puerto 8888), no `npm run dev`.
- **No** `git commit`/`push` ni deploy a Netlify sin permiso explícito del usuario.
- Cada `netlify deploy --prod` DEBE incluir `--message`.
- Un cambio en producción a la vez.
