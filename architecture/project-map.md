# Project Map — MaldoxFilm

> Documento doble propósito:
> 1. **Índice maestro del SCP** (§0): qué información hay en cada documento.
> 2. **Mapa del código** (§1+): dónde está implementada cada funcionalidad.
>
> Raíz del código: `/Users/angelomaldonado/Desktop/maldoxfilm-vue/`

---

## 0. Índice maestro del SCP

Catálogo de documentos. Cada concepto tiene **un único hogar canónico**; el resto apunta a él.

### Raíz
| Documento | Qué información contiene | Cuándo leerlo |
|---|---|---|
| `README.md` | Propósito del SCP, política de carga inteligente, reglas de mantenimiento | Al empezar a usar el SCP |
| `project-memory.md` | **Estado vigente**, decisiones activas, riesgos, próximos pasos, mapa rápido de "dónde tocar" | Inicio de sesión / "¿en qué vamos?" |
| `current-sprint.md` | Tablero del sprint: ✅ completado / 🔄 en progreso / ⏳ pendiente / ⚠ riesgos | Para ver avance del sprint |
| `coding-standards.md` | Convenciones, nomenclatura, capas (puro vs efectos), patrones obligatorios, reglas de testing/git/deploy | Antes de escribir código |

### `architecture/`
| Documento | Qué información contiene | Cuándo leerlo |
|---|---|---|
| `project-map.md` (este) | Índice del SCP (§0) + mapa del código: vistas, componentes, composables, servicios, stores, functions, tests, puntos de entrada de debug (§1+) | "¿Dónde está X?" |
| `business-rules.md` | El **"por qué"**: reglas funcionales y de negocio (selección de fuente RD, `rdId`, audio español/subs, secciones del catálogo, progreso, Mi Lista, "Volver"→sección/género previos) | "¿Por qué funciona así?" |
| `environment.md` | Entornos (DEV/PROD), **puertos** (8888/5173), **variables** (RD_TOKEN), URLs/endpoints externos, almacenamiento local | Setup local / configuración |
| `frontend.md` | Arquitectura Vue: capas, bootstrap (`main.ts`), inyección de dependencias, estilos, cancelación async | Trabajo general de frontend |
| `routing.md` | Tabla de rutas Vue Router, navegación clave, género en URL, overlay de "Ver todo" | Rutas / navegación |
| `state-management.md` | Stores Pinia (`player`, `progress`, `myList`, `device`) + persistencia localStorage | Estado de la app |
| `components.md` | Catálogo de componentes (catalog/layout/player) con props/eventos | UI / componentes |
| `player.md` | **Reproductor** (módulo más complejo): cascada de motores, transcode, audio multi-pista, subtítulos, troubleshooting | Cualquier tema del reproductor |
| `integrations.md` | APIs externas: TMDB, Torrentio, Real-Debrid, OpenSubtitles, iframes, libs CDN, Netlify Functions | Integraciones de red |
| `realdebrid-api.md` | **Referencia API Real-Debrid v1.0**: base URL, auth, endpoints usados, códigos de error, estados, rate limit | Trabajar con la API de RD |
| `deployment.md` | Cómo correr (`netlify dev`) y desplegar (Netlify), `netlify.toml`, build, reglas de deploy | Levantar/desplegar |
| `security.md` | **Política del RD_TOKEN** (server-side only, token comprometido, Fase 1/2), claves TMDB/OpenSubtitles, XSS subtítulos | Cualquier tema de secretos/seguridad |

### `adr/` y `sessions/`
| Documento | Qué información contiene |
|---|---|
| `adr/ADR-001.md` | Decisión: arquitectura por capas (puro vs efectos) + DI de clientes |
| `adr/ADR-002.md` | Decisión: RD_TOKEN server-side vía Netlify Function |
| `adr/ADR-003.md` | Decisión: "Ver todo" overlay fullscreen + género en URL |
| `adr/ADR-004.md` | Decisión: RD server-side (addMagnet→transcode) para no-cacheado, sin Torrentio |
| `adr/ADR-005.md` | Decisión: subtítulos = OpenSubtitles (RD no expone pistas) |
| `adr/ADR-006.md` | **Propuesta**: borrar torrents de RD al cerrar el reproductor |
| `adr/ADR-007.md` | Decisión: barra del navegador webOS — `requestFullscreen` al volver + nav `position:sticky` |
| `adr/ADR-008.md` | **PROPUESTA**: RD-mediaInfos como pipeline canónico + atajo directo-Range (a implementar 2026-06-14) |
| `sessions/YYYY-MM-DD.md` | Bitácora por fecha (trabajo, decisiones, problemas, pendientes del día) |

### Búsqueda rápida: "necesito saber sobre…"
| Necesito… | Documento(s) |
|---|---|
| Estado actual / qué sigue | `project-memory.md`, `current-sprint.md` |
| Dónde está implementado X (archivo) | `project-map.md` §1+ |
| Reproductor / RD / DASH / subtítulos | `player.md` (+ `project-map.md`) |
| Por qué una regla funcional | `business-rules.md` |
| APIs externas (TMDB/RD/Torrentio/OpenSubtitles) | `integrations.md` |
| Token RD / secretos | `security.md` (+ `ADR-002`) |
| Puertos / variables / cómo correr | `environment.md`, `deployment.md` |
| Rutas / navegación | `routing.md` |
| Estado/stores (Pinia) | `state-management.md` |
| Componentes UI | `components.md` |
| Convenciones de código | `coding-standards.md` |
| Por qué se decidió algo | `adr/` |

---

## 1. Punto de entrada y bootstrap

```
index.html                → carga Google Fonts (Oswald/Roboto) + monta /src/main.ts
src/main.ts               → bootstrap(): Pinia → deviceStore.detect() → createAppServices()
                            (async, resuelve RD token) → app.provide(APP_SERVICES_KEY) →
                            router → mount('#app')
src/App.vue               → layout: Nav según dispositivo (Desktop/Mobile/TV) + <router-view>
src/services/bootstrap.ts → createAppServices(): arma clientes TMDB/Torrentio/RD +
                            RdStreamResolver; fetchRdToken() pide el token a la Netlify Function.
```

---

## 2. Flujo general de la app

```
Catálogo (TMDB) ──► Carruseles/Grillas ──► Detalle ──► Reproductor
                                                          │
                                          ┌───────────────┴───────────────┐
                                          ▼                               ▼
                                    Real-Debrid (RD)                Iframe fallback
                                  (video nativo + subs)          (UnlimPlay / vidlink)
```

1. **Catálogo**: views piden listas a TMDB (`useTmdbList`) y las pintan en `Carousel`/grillas.
2. **Detalle** (`DetailView`): metadata + acciones (Reproducir / Mi Lista / Volver).
3. **Reproductor** (`PlayerView` → `VideoPlayer`): resuelve el mejor stream RD
   (`rdStream` + `streamSelector`), reproduce con cascada HEVC/DASH(Shaka)/HLS(hls.js)/directo,
   inyecta subtítulos (`useSubtitles`). Si RD falla → iframe.

---

## 3. Vistas (rutas) — `src/views/`

| Vista | Ruta | Qué hace |
|---|---|---|
| `HomeView.vue` | `/` | Inicio: hero + tabs + géneros + Continuar viendo + carruseles + TOP 10 |
| `MoviesView.vue` | `/peliculas` | Películas: cabecera + géneros + 5 secciones por género + "Ver todo" |
| `SeriesView.vue` | `/series` | Series: ídem con ids de TV |
| `AnimeView.vue` | `/anime` | Anime (TMDB con keyword anime) |
| `DetailView.vue` | `/pelicula/:id`, `/serie/:id/:s/:e` | Detalle + acciones |
| `PlayerView.vue` | `/ver/:type/:id` | Reproductor (topbar, selector de fuente, iframe; delega RD a `VideoPlayer`) |
| `SearchView.vue` | `/buscar?q=` | Resultados de búsqueda |
| `MyListView.vue` | `/mi-lista` | Mi Lista (localStorage) |
| `AllResultsView.vue` | `/ver-todo?title=&endpoint=&type=` | Grilla paginada (scroll infinito, fondo telón) |

Router: `src/router/index.ts`. Detalle en `architecture/routing.md`.

---

## 4. Componentes — `src/components/`

### `catalog/`
- `Carousel.vue` — fila horizontal de `MovieCard` + flechas + "Ver todo".
- `MovieCard.vue` — tarjeta póster (rating circle, overlay, progreso).
- `HeroBanner.vue` — banner rotativo del Inicio (trending/movie/day, 7s).
- `ContinueWatching.vue` — sección "Continuar viendo" (progressStore).
- `TopNumbered.vue` — fila TOP 10 con números grandes estilo Netflix.
- `GenreFilter.vue` — barra de pills de género (delega a `TvGenreBar` en TV).
- `TvGenreBar.vue` — **menú de géneros estilo Netflix (solo TV)**: activo anclado a la izquierda,
  deslizamiento, **scroll infinito** (lista triplicada), filtro al asentarse, navegación D-pad.
- `SearchBar.vue` — input de búsqueda (debounce 400ms, emite search/clear/submit).

### `layout/`
- `NavDesktop.vue` — barra superior desktop (logo, links, buscador overlay, perfil).
- `NavMobile.vue` — nav inferior móvil.
- `NavTV.vue` — sidebar/topnav TV.

### `player/`
- `VideoPlayer.vue` — orquesta RD (usa `usePlayer`), controles, panel ⚙️, overlay de subs.
- `PlayerSettingsPanel.vue` — panel Audio/Subtítulos/Velocidad + offset de subs.
- `SubtitleOverlay.vue` — render del cue actual sobre el video.
- `SubtitleControls.vue` — (legado, ahora el offset vive en el panel ⚙️) — **huérfano**.
- `SourceSelector.vue` — botones RD/UnlimPlay/vidlink.

Detalle en `architecture/components.md`.

---

## 5. Composables — `src/composables/` (orquestación)

| Composable | Responsabilidad |
|---|---|
| `usePlayer.ts` | **Núcleo del reproductor RD**: cascada HEVC→DASH(Shaka)→HLS(hls.js)→directo, transcode, probe audio español, watchdogs, cambio de pista, recuperación de stall, limpieza de torrent (ADR-006). |
| `useTvSpatialNav.ts` | **Navegación D-pad/Magic remote en TV**: ←/→ por cards del carrusel, ↓/↑ entre categorías, foco sigue al puntero. Montado en `App.vue` (solo TV). |
| `useSubtitles.ts` | Búsqueda/descarga/parseo/inyección de subtítulos (OpenSubtitles + functions). |
| `useNetflixControls.ts` | Barra de controles custom (play/seek/volumen/velocidad). |
| `useFullscreen.ts` | Fullscreen custom + orientación. **En TV NO toca el Fullscreen API nativo** (ni `requestFullscreen` ni `exitFullscreen`): tocarlo muestra/oculta la barra del navegador webOS y rompía el nav. El overlay CSS del player ya cubre la pantalla. (2026-06-13) |
| `useTmdbList.ts` | Cargar una lista TMDB (`load(endpoint)` → items). **Cache stale-while-revalidate**: `memCache` (sesión) + `safeStorage`/localStorage (prefijo `tmdblist:`, TTL 6h). Lista ya vista se pinta al instante SIN skeleton y revalida en 2º plano → no más "tarjetas grises" al re-montar HomeView tras volver del player. (2026-06-13) |
| `useGenreCatalog.ts` | Barra de géneros + reescritura de títulos (usado por Anime; Movies/Series ahora gestionan género local). |
| `useToast.ts` | Toasts (singleton). |
| `useAppServices.ts` | `inject(APP_SERVICES_KEY)` → clientes de red. |

---

## 6. Servicios — `src/services/` (lógica pura + clients)

| Servicio | Responsabilidad |
|---|---|
| `bootstrap.ts` | Arma clientes de red + RdStreamResolver; `fetchRdToken()`. |
| `tmdb.ts` | Cliente TMDB (`get`, `getImdbId`). |
| `torrentio.ts` | Cliente Torrentio (lista de streams de un torrent). |
| `realdebrid.ts` | Cliente RD (`resolveProxyUrl`, `fetchDownloads`, `fetchTranscode`). |
| `rdStream.ts` | **Orquestación de `rdGetStream`**: IMDB→Torrentio→scoring→resolve→match downloads→ensamblar `SelectedStream`. |
| `streamSelector.ts` | **PURO**: scoring/ranking (`selectBestStream`), matching (`resolveActiveStream`), ensamblado (`buildSelectedStream`). |
| `subtitles.ts` | PURO: parseo SRT, scoring de subtítulos, helpers. |
| `playerSources.ts` | Fuentes iframe (UnlimPlay/vidlink), `RD_SRC_IDX=99`. |
| `playback.ts` | Helpers de progreso/auto-next (puros). |
| `catalog.ts` | Helpers de catálogo (poster URL, géneros, score circle, títulos por género). |
| `progress.ts` | Lógica de "continuar viendo" (puro) + claves localStorage. |
| `myList.ts` | Lógica de Mi Lista (puro). |
| `navigation.ts` | Ítems de nav + mapeo de rutas. |
| `deviceDetect.ts` | Detección TV/desktop/mobile. |
| `fullscreen.ts` | Lógica pura de fullscreen (paths de icono, estilos, orientación). |
| `anime1.ts` | Fuente especial Anime1V. |
| `safeStorage.ts` | Wrapper localStorage seguro. |
| `lruCache.ts` | Cache LRU. |

---

## 7. Stores — `src/stores/` (Pinia)

| Store | Estado |
|---|---|
| `player.ts` | `current` (PlayerState), `generation`, `sourceIndex` (RD_SRC_IDX=99 por defecto), `isActive`. |
| `progress.ts` | `all` (mapa de progreso) → localStorage `sx_prog_v2`. |
| `myList.ts` | Mi Lista → localStorage `sx_mylist`. |
| `device.ts` | Modo de dispositivo (tv/desktop/mobile). |

Detalle en `architecture/state-management.md`.

---

## 8. Netlify Functions — `netlify/functions/`

| Función | Qué hace |
|---|---|
| `rd-token.js` (ESM) | Devuelve `{token}` desde `process.env.RD_TOKEN`. |
| `rd-stream.js` (ESM) | No-cacheado server-side (ADR-004): addMagnet→selectFiles→unrestrict→transcode DASH. |
| `rd-cleanup.js` (ESM) | Borra el torrent de `rd-stream` al cerrar/cambiar (ADR-006). |
| `subhash.cjs` (CommonJS, verbatim del original) | Proxy a OpenSubtitles `/download` (devuelve link/remaining/file_name) + hash OS por rdId. |
| `offset.cjs` (CommonJS) | Crowdsourcing de offsets de subtítulos (Netlify Blobs). |

Detalle en `architecture/integrations.md` y `architecture/deployment.md`.

---

## 9. Tests — `tests/` (Vitest, 251 tests)

Cubren los servicios puros: `streamSelector`, `rdStream`, `subtitles`, `catalog`,
`progress`, `myList`, `navigation`, `deviceDetect`, `fullscreen`, `playback`,
`playerSources`, `apiClients`, `stores`, `useNetflixControls`, `useToast`, `anime1`, `lruCache`.

---

## 10. Puntos de entrada críticos (para debug)

| Síntoma | Empezar por |
|---|---|
| No reproduce / cae a iframe | `usePlayer.ts` (loadRdSource) + logs `[RD]` |
| No hay subtítulos | `useSubtitles.ts` + `VideoPlayer.onStreamReady` + logs `[SUB]` |
| Token RD | `bootstrap.ts` (fetchRdToken) + `.env` + `netlify dev` |
| Audio equivocado (inglés vs español) | `usePlayer.ts` (probeSpanishDashTrack, isDualLatFilename) |
| Inicio mal | `HomeView.vue` |
| Filtros género | `MoviesView.vue` / `SeriesView.vue` + `GenreFilter.vue` |
| Scroll atrapado | `Carousel.vue` (overflow-y) |
