# Components — MaldoxFilm

> Catálogo de componentes. `src/components/`. Patrón: props in, eventos out, scoped CSS.

---

## catalog/

### `Carousel.vue`
Fila horizontal de `MovieCard` con flechas (blancas, grandes, estilo Netflix) y
"VER TODO ›" opcional (`seeAllTo`). Props: `title`, `subtitle?`, `items`,
`fallbackType`, `loading?`, `seeAllTo?`, `isTvMode?`. Emite `select`.
`overflow-y:hidden` + padding 52px.

### `MovieCard.vue`
Tarjeta póster 2:3. Rating circle (SVG dasharray), overlay con título/año/rating,
barra de progreso opcional. Props: `item`, `fallbackType`, `isTvMode?`, `progressPct?`,
`progressLabel?`. Emite `select` con `{id, type}` (resuelve type con `media_type` o fallback).

### `HeroBanner.vue`
Banner rotativo del Inicio: `trending/movie/day` (8 ítems, rota cada 7s). Backdrop +
título + meta + sinopsis + botones Reproducir/Más Info (ambos → detalle). Sin dots
(se quitaron a pedido).

### `ContinueWatching.vue`
Sección "Continuar viendo": `progressStore` filtrado (`pct>3 && <97`, top 20 por `ts`),
trae posters TMDB en paralelo. Tarjeta compacta con barra de progreso + ✕ quitar +
"Limpiar historial". Solo se muestra si hay ítems.

### `TopNumbered.vue`
Fila TOP 10 con números grandes estilo Netflix (oscuro + contorno gris, el póster
solapa). Props: `typeLabel`, `when?`, `items`, `loading?`, `fallbackType`, `seeAllTo?`.

### `GenreFilter.vue`
Barra de pills de género (scroll horizontal, padding 52px). Props: `options`
(`{id,label}[]`), `activeId`. Emite `select(id)`.

### `SearchBar.vue`
Input con debounce 400ms. Emite `search` (debounce o Enter), `clear` (vacío), `submit`
(Enter). Prop `autofocus`. Usado inline en el overlay de búsqueda del nav.

---

## layout/

- **`NavDesktop.vue`** — barra superior: logo Oswald (MALDOX azul + FILM blanco),
  links (pill en hover, azul sin subrayado activo), buscador **pill → overlay
  centrado** (Teleport al body, × y Esc para cerrar), perfil. Fondo negro sólido, sticky.
- **`NavMobile.vue`** — nav inferior móvil.
- **`NavTV.vue`** — topnav TV (logo + Inicio/Películas/Series/Anime/Mi Lista + Buscar/Fullscreen).
  ⚠️ Usa **`position: sticky`** (NO `fixed`): debe SOBREVIVIR al Fullscreen API que el player usa
  en TV para ocultar la barra del navegador webOS — un `position:fixed` dentro de un `:fullscreen`
  deja de pegarse al scrollear (bug webkit) y el nav "desaparecía". Como es el primer elemento del
  flujo, queda pegado en `top:0` igual que `fixed`. `App.vue` ya NO pone `padding-top:76px` en TV
  (el nav sticky ocupa ese espacio en el flujo). (2026-06-13)

---

## player/

### `VideoPlayer.vue`
Orquesta el camino RD. Usa `usePlayer` (carga), `useSubtitles`, `useFullscreen`,
`useNetflixControls`. Callbacks de `usePlayer`:
- `onStarted` → emite `started`, attach controles.
- `onStreamReady({selected, hasNativeSpanish, spanishTrack})` → setea panel
  (`audioPanelReady`), toggle de subs, y **fetchAndInject** subs si `!hasNativeSpanish && imdbId`.
- `onNativeSpanishDetected`, `onAudioTrackChanged`, `onFallbackToNextSource`, etc.
Click en el `<video>` = play/pausa (cierra panel si está abierto). Botón fullscreen
reactivo (`:d` según `isFullscreen`).

### `PlayerSettingsPanel.vue`
Panel ⚙️ con 2 vistas: Audio+Subtítulos / Velocidad. Incluye la fila de **offset de
subtítulos** (−5s/−1s/reset/+1s/+5s), visible solo si subs ON. Filas de audio según
3 casos de `_spInit` (solo español / dual / solo inglés).

### `SubtitleOverlay.vue`
Render del cue activo sobre el video (texto plano + `white-space:pre-line`, sin
`v-html` por seguridad — los .srt son contenido no confiable).

### `SubtitleControls.vue`
**Huérfano** (ya no se usa; el offset se movió al panel ⚙️). Candidato a borrar.

### `SourceSelector.vue`
Botones RD / UnlimPlay / vidlink.

---

## Otros

- `Toast.vue` — toasts (singleton `useToast`).
- `HelloWorld.vue` — scaffold de Vite, **sin uso** (candidato a borrar).
