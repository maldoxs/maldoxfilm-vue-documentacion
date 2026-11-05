# Routing — MaldoxFilm (Vue Router 4)

> `src/router/index.ts`. History mode (`createWebHistory`).

---

## Rutas

| Path | name | Componente | Props |
|---|---|---|---|
| `/` | home | `HomeView` | — |
| `/peliculas` | movies | `MoviesView` | lee `?genero=` (estado local) |
| `/series` | series | `SeriesView` | lee `?genero=` |
| `/anime` | anime | `AnimeView` | — |
| `/pelicula/:id` | movie-detail | `DetailView` | `id`, `type='movie'` |
| `/serie/:id/:s/:e` | series-detail | `DetailView` | `id`, `type='tv'`, `season`, `episode` |
| `/ver/:type/:id` | player | `PlayerView` | `type`, `id` (+ `?s=&e=` para tv) |
| `/mi-lista` | mylist | `MyListView` | — |
| `/buscar` | search | `SearchView` | `query` (de `?q=`) |
| `/ver-todo` | all-results | `AllResultsView` | `title`, `endpoint`, `type` (de query) |
| `/:pathMatch(.*)*` | — | redirect `/` | catch-all |

---

## Navegación clave

- **Seleccionar título** (cards): `router.push('/pelicula/:id')` o `/serie/:id/1/1`.
- **Reproducir**: desde detalle → `/ver/movie/:id` o `/ver/tv/:id?s=&e=`.
- **"Volver" del detalle** (`DetailView.goBack`): `router.back()` → vuelve a la sección/género
  de donde vino el usuario (p.ej. `/peliculas?genero=27`); el género viaja en la URL, así que el
  filtro se mantiene. Fallback a `router.push('/')` SOLO si no hay historial previo
  (`window.history.state.back == null`, deep-link/entrada directa). *(Cambiado 2026-06-13 — antes
  iba SIEMPRE a Inicio; el usuario pidió que conserve sección + filtro.)*
- **Sección activa del nav**: `App.vue` deriva `activeKey` del nombre de ruta; en rutas que NO son
  sección (`movie-detail`/`series-detail`/`player`/`all-results`) usa `lastSection` (última sección
  visitada) → el nav se queda en "Películas"/"Series"/… en vez de saltar a "Inicio" (2026-06-13).
- **"Ver todo"** (carruseles de Movies/Series): navega a
  `/ver-todo?title=...&endpoint=<TMDB endpoint url-encoded>&type=movie|tv`.
  El endpoint completo viaja url-encoded en el query `endpoint`.
- **Filtro de género** (Movies/Series): `router.replace({query:{genero:id}})` — así
  "Volver" desde la grilla restaura el género activo. Se inicializa leyendo
  `route.query.genero` en `setup()`.

---

## Notas de scroll/overlay

- `AllResultsView` es un **overlay fixed fullscreen** con su propio scroll
  (`position:fixed; inset:0; overflow-y:auto; z-index:600`) — para que la cabecera
  `sticky` quede fija y no la tape el nav. Tapa el nav mientras está abierta.
- `scrollBehavior` del router: ver `src/router/index.ts`.
