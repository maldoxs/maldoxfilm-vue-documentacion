# Business Rules — MaldoxFilm

> Reglas funcionales, lógica de negocio, restricciones y conceptos.
> Responde: **¿Por qué existe esta implementación?**

---

## 1. Concepto del producto

MaldoxFilm es un agregador de streaming: usa **TMDB** para el catálogo y metadata,
**Torrentio** para descubrir streams de un título, y **Real-Debrid (RD)** para
convertir esos torrents (ya cacheados) en URLs HTTP reproducibles en HD. Si RD no
puede entregar el contenido, cae a reproductores **iframe** de terceros
(UnlimPlay, vidlink) como respaldo.

Usuario objetivo: hispanohablante (Chile). **Jerarquía de audio (regla central):**
**Latino > Inglés + subtítulos español.** El castellano (España) NO es opción —
se trata igual que inglés. Si hay latino → audio latino; si no → inglés + subs ES.

---

## 2. Selección de fuente del reproductor

- Cada título arranca SIEMPRE intentando **Real-Debrid** (`RD_SRC_IDX = 99`), que es
  el `sourceIndex` por defecto. Esto preserva `_defaultSrcIdx = RD_SRC_IDX` del original.
- `playerStore.load()` (al abrir un título) **resetea `sourceIndex` a RD_SRC_IDX** —
  así un título nuevo no arrastra la fuente del anterior (bug histórico ya corregido).
- Fuentes iframe: `SOURCES[0] = UnlimPlay`, `SOURCES[1] = vidlink`.
- Si RD no entrega URL reproducible → `fallback-to-next-source` → UnlimPlay → (timeout) → vidlink.
- El cambio MANUAL de fuente (botones) usa `switchSource(idx)` y emite un toast;
  el auto-fallback interno usa `fallbackToFirstSource()` (SIN toast extra, para no
  duplicar mensajes — el original nunca pasaba por `switchSource` en el fallback).

---

## 3. Resolución del stream RD (`rdGetStream`)

Orden EXACTO de 5 pasos (`services/rdStream.ts`):
1. **IMDB ID** desde TMDB external_ids.
2. **Streams** desde Torrentio.
3. **Scoring/ranking** → mejor stream (`selectBestStream`, `scoreStream`). Pesos de idioma
   (P6, actualizado 2026-06-24): **Latino +200** · **Inglés +5/+10** · **Castellano = Inglés
   (sin bonus)**. Premia 1080p, H264/AAC; penaliza HEVC/4K/audio incompatible. El +200 al
   latino hace que un Dual-Lat de Cinecalidad gane sobre cualquier YIFY inglesa.
4. **Resolver proxy** Torrentio → URL CDN real de RD.
5. **Match en downloads** de RD para obtener `rdId` (+ 2 rondas de fallback). El
   `rdId` es necesario para el **pipeline /t/** y el transcode DASH/HLS.
   - **Preferencia latino server-side**: si el match NO tiene latino pero hay un
     Dual-Lat `[RD+]` en el pool, se resuelve server-side (`rd-stream`) para forzar
     su descarga. SOLO busca latino (no castellano).

### Concepto clave: `rdId`
- Es el id del archivo en la cuenta RD (aparece en la URL CDN `/d/{rdId}/...`).
- Sin `rdId` no se puede pedir transcode → solo queda reproducción directa, que
  falla si el navegador no soporta el códec → cae a iframe.
- **Si el RD_TOKEN está inválido/expirado**, `/downloads` no responde → "Sin match
  en downloads — rdId null" → iframe. (Es problema de token, no de código.)

---

## 4. Cascada de reproducción (motor de video) — ACTUALIZADA 2026-06-24

**Orden real:**
1. **Direct Play** (MP4/H264/AAC) → seek nativo instantáneo + calidad original.
2. **Pipeline /t/** (CDN privada de RD) → far-seek universal + audio AAC. Para los que
   NO son Direct Play (MKV/x265). Genera DASH en cualquier posición (`full.mpd?t={seg}`).
3. **Transcode DASH/HLS legacy** → último recurso con rdId.
4. **Iframe** (UnlimPlay/vidlink) → sin rdId o todo lo anterior falló.

**Toast "Seek fluido"** al iniciar: indica el camino (`✅ Seek fluido` / `✅ Seek fluido · Audio Latino`).

> Detalle técnico del pipeline /t/ (resolveTpipeline, pingSeek, waitForSegmentAt, offset)
> y motores → `player.md`. No duplicar aquí.

### Anime (regla de producto, decidida 2026-06-24)
**Anime SIEMPRE usa Anime1V — NUNCA pasa por RD.** RD para anime es errático (raws
japoneses sin subs). Anime1V está hecho para anime: SUB (japonés + subs español 🇯🇵) y
DUB (latino 🇲🇽) de forma consistente.
- `loadActiveSource`: si `isAnime` → `sourceIndex=0` (no RD) → iframe Anime1V directo.
- **Una sola carga**: HLS si existe (default), si no StreamiX. Nunca doble carga.
- Selector consistente: SUB/DUB · StreamiX · HLS · ... filtrado por idioma. Latino en DUB.
- Volver navega DIRECTO al detalle (`router.push`), no `router.back()` — cambiar `iframe.src`
  ensucia el historial y `back()` caía en iframe blanco (doble click).

---

## 5. Detección de audio español y subtítulos

- `_hasNativeSpanish` (original) = `true` SOLO cuando el **probe DASH confirma** una
  pista de audio en español real (`probeSpanishDashTrack` hace HEAD a la URL de la
  pista candidata). En ese caso NO se piden subtítulos (audio ya en español).
- Si NO está confirmado el español → se inyectan **subtítulos ES** (OpenSubtitles).
- `isDualLatFilename(nombre)` detecta "dual/multi/lat/spa/español/castellano/latino"
  en el nombre — se usa para el toggle inicial de subs del panel, NO como verdad
  absoluta del audio.
- **PUNTO ABIERTO**: la detección de "audio español" por nombre de archivo no es fiable
  para multi-audio (puede sonar inglés). Detalle del problema, el caso "Weapons" y su
  estado → **`player.md` §6** (única fuente).

### Subtítulos (OpenSubtitles)
- Búsqueda por **release name** → fallback por **IMDB ID**.
- Scoring de subtítulos; descarga del `.srt` vía Netlify Function `subhash`
  (con hasta 4 reintentos, validando ≥100 cues).
- Offset de sincronización: local (localStorage) + **crowdsourced** (function `offset`
  con Netlify Blobs).
- Quirk preservado: tras inyectar, el status muestra siempre `✅ ES | offset: 0s`.

---

## 6. Catálogo / secciones

### Inicio (`/`)
Hero rotativo (8 destacados de `trending/movie/day`, rota 7s) → tabs
(Películas/Series/Populares, controlan los 4 carruseles dinámicos) → barra de
géneros (13 opciones) → Continuar viendo → carruseles → TOP 10 (Películas/Series).

### Películas/Series (`/peliculas`, `/series`)
Al elegir un género, las **5 secciones** se reescriben:
`🔥 {género} más populares ahora`, `⭐ más vistas`, `🏆 mejor puntuadas`,
`🎬 recientes (últimos 2 años)` / `📡 en emisión`, `🗓️ próximos`.
Endpoints `/discover/movie|tv?with_genres=...&sort_by=...`. El género se persiste en
la URL (`?genero=`) para que "Volver" desde la grilla mantenga el filtro.
**Series usa ids de TV** (Acción=10759, Sci-Fi=10765, etc.), distintos de cine.

### "Ver todo" → grilla paginada (`/ver-todo`)
Overlay fullscreen con scroll infinito (paginación TMDB), pósters grandes y fondo
"telón" (backdrop borroso). El contador ("X resultados") es el `total_results` real
de TMDB; muestra ~20 por página y carga más al bajar.

---

## 7. Continuar viendo / progreso — ACTUALIZADA 2026-06-24

- Se guarda progreso por título (`sx_prog_v2`): `{id, type, pct, positionSec, runtimeMin,
  season, episode, title, ts}`.
- **`positionSec`** (segundos reales, no % estimado): se guarda cada 15s y al cerrar.
  Con pipeline /t/ usa `tpipelineOffset + video.currentTime`. Al reabrir, **restaura la
  posición** automáticamente (Direct Play vía `currentTime`, /t/ vía `tpipelineSeekTo`).
- **`runtimeMin`** (duración TMDB): para mostrar "1h 38min" en tarjetas/ContinueWatching.
- "Continuar viendo" muestra ítems con `pct > 3 && pct < 97`, ordenados por `ts` desc, máx 20.
- Botón "Reproducir" del detalle: "Continuar T1·E2" (series) / "Continuar · 30 de 120 min" (pelis).

---

## 8. Mi Lista

- localStorage `sx_mylist`. Formato MIXTO preservado del original: strings sueltos
  (legado, solo id) **y** objetos `{id,type,title,poster}`. No romper datos existentes.

---

## 9. Navegación "Volver" (decisión de producto) — ACTUALIZADA 2026-06-13

El botón "← Volver" del detalle vuelve a la **página anterior** (`router.back()`),
preservando la sección y el filtro de donde vino el usuario (p.ej. Películas → género Terror →
ficha → Volver → `/peliculas?genero=27`, con Terror aún filtrado porque el género viaja en la URL).
Fallback a Inicio (`router.push('/')`) SOLO si no hay historial previo (deep-link/entrada directa,
detectado con `window.history.state.back`).

> **Cambio de decisión:** antes iba SIEMPRE a Inicio (se priorizaba predecibilidad). El usuario
> pidió explícitamente (2026-06-13) que "Volver" conserve la sección + género — alineado con el
> comportamiento del original (`_navState`). El nav también mantiene la sección activa en la ficha
> (`lastSection` en `App.vue`), ya no salta a "Inicio".

### Volver desde el REPRODUCTOR — ACTUALIZADA 2026-06-24
El botón "← Volver" del reproductor (`closePlayer`) navega **directo al detalle** del título
(`router.push('/pelicula/:id'` o `/serie/:id/:s/:e'`), NO `router.back()`. Motivo: cambiar
`iframe.src` (anime/iframe) agrega una entrada al historial del navegador → `back()` caía
primero en el reproductor con iframe en blanco (requería dos clicks). `closePlayer` usa
`bumpGeneration()` (no `close()`) para no resetear `sourceIndex` mientras está montado
(evita el parpadeo del reproductor RD al volver). El `close()` completo lo hace `onBeforeUnmount`.

---

## 10. Dispositivos

Detección TV/desktop/mobile en `deviceDetect.ts` → `deviceStore`. Cambia el nav
(NavDesktop/NavMobile/NavTV) y tamaños (TV usa pósters/paddings mayores).
