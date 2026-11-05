# Player — MaldoxFilm (el módulo más complejo)

> Reproductor Real-Debrid: cascada de motores, transcode, audio multi-pista,
> subtítulos. Núcleo en `src/composables/usePlayer.ts` (orquestación) +
> `src/services/streamSelector.ts` / `rdStream.ts` (selección pura).

---

## 1. Capas

```
PlayerView.vue       topbar, selector de fuente, iframe (UnlimPlay/vidlink/Anime1V), auto-hide
   └─ VideoPlayer.vue   camino RD: controles, panel ⚙️, overlay subs, freeze-frame /t/
        └─ usePlayer.ts    cascada DirectPlay→/t/→transcode→iframe, watchdogs
        └─ useSubtitles.ts subtítulos (OpenSubtitles + functions, timeOverride para /t/)
rdStream.ts          orquesta rdGetStream (IMDB→Torrentio→scoring→resolve→match)
streamSelector.ts    PURO: scoring/matching/ensamblado (testeado)
rd-t-pipeline.ts     cliente pipeline /t/ (resolveTpipeline, pingSeek, buildMpdUrl, ...)
```

`RD_SRC_IDX = 99` (RD por defecto). `SOURCES[0]=UnlimPlay`, `SOURCES[1]=vidlink`.
**Anime**: `sourceIndex=0` forzado → nunca RD → Anime1V (SUB/DUB/HLS).
**279 tests verdes**, `vue-tsc -b` OK (Netlify usa `-b`, más estricto que `--noEmit`).

---

## 2. Resolución del stream (`rdStream.getStream`)

5 pasos (orden EXACTO):
1. IMDB ID desde TMDB.
2. Streams desde Torrentio (`fetchStreams`).
3. `selectBestStream` (scoring): 1080p, español, H264/AAC; penaliza audio malo.
4. `resolveProxyUrl(best.url)` → URL CDN real de RD.
5. `resolveActiveStream`: match contra `/downloads` → `rdId` (+ 2 rondas fallback).
   Si no hay match → `rdId=null` y se setea `unavailableInRd`.

**Capas de resolución (prioridad):**
1. **Cacheado** — `rdId` directo desde `/downloads`.
2. **Rescate del pool** — recorre TODO el pool y usa la mejor versión ya cacheada
   (con rdId). No escribe en RD. (Resuelve el caso "Weapons".)
3. **Server-side `rd-stream` (ADR-004)** — si NO hay rdId ni versión cacheada:
   `rdStream.ts` recoge los top infoHashes y llama a `serverResolve` (Function
   `rd-stream`); puebla `serverDashUrl`/`serverLiveMp4Url`/`serverHlsUrl`/`serverDirectUrl`.
   `usePlayer.loadRdSource` los reproduce con Shaka directo. *Escribe en RD (addMagnet).*
4. **Iframe** — último recurso.

Devuelve `SelectedStream`: `{url, rdId, isX265, fallbackUrl, imdbId, streamFilename,
hasAAC, rdDownloadUrl, rdFilesize, infoHash, unavailableInRd,
serverDashUrl?, serverLiveMp4Url?, serverHlsUrl?, serverDirectUrl?}`.

> El `serverResolve` es **inyectable** (default no-op en `rdStream.ts`; real en
> `bootstrap.ts` → `fetch('/.netlify/functions/rd-stream?infoHash=...')`). Por eso los
> tests no tocan la red y siguen 251/251.

**Logs `[RD]`** (console.warn) en cada paso — restaurados 1:1 del original, NO borrar.

---

## 3. Cascada de reproducción (`usePlayer.loadRdSource`) — ACTUALIZADA 2026-06-24

Orden REAL:

0. **Anime** (`opts.isAnime?.()`): fallback INMEDIATO a iframe Anime1V. RD nunca se usa
   para anime (raws japoneses). Ver `business-rules.md` §4.
1. **Guards**: sin `streamUrl` → fallback iframe. `unavailableInRd` → **fallback inmediato**
   (antes solo toast). x265 → toast "Optimizando...".
1.b **VERIFICACIÓN DEL CÓDEC REAL (2026-08-02, ADR-010) — decide esto, no el nombre.**
   Con `rdId` se consulta SIEMPRE `/streaming/mediaInfos/{rdId}` y ese dato manda sobre la
   heurística de nombre, en AMBAS direcciones:
   - nombre OK + audio real AC3 → NO Direct Play (ya existía, ADR-009 fix 3)
   - nombre "MKV sin AAC" + archivo real h264+aac → **SÍ Direct Play** (nuevo; es lo que
     resolvió los cortes: se mandaba a convertir un archivo que ya servía)
   - sin dato (`null`) → heurística por nombre, comportamiento previo intacto
   `canTryDirect = videoSirve && audioSirve`. Helpers: `hasNativeDecodableAudio` y
   `videoNeedsHevcSupport` (`services/mediaInfos.ts`).

2. **Direct Play**: `tryHevcDirectPlay`. Si reproduce → `onStreamReady` + monitor de stall
   + termina. Muestra el aviso "⚡ Reproducción directa" (condición: `isDirectPlay`).
   Antes de asignar `src` se espera a que Shaka suelte el `<video>` (`_shakaDetachAndWait`):
   con el MediaSource enganchado la carga no arranca nunca. La paciencia se renueva mientras
   llegue data (`progress`), tope `DIRECT_PLAY_MAX_WAIT_MS` = 30 s — los 10 s fijos no
   alcanzaban para el índice de un MP4 de ~3 GB.
3. **Pipeline /t/** (`if rdId`, ANTES del transcode legacy): `tryTpipeline`. CDN privada de
   RD genera DASH en cualquier posición (`full.mpd?t={seg}`) → **far-seek universal + audio
   AAC**. Flujo: `resolveTpipeline(rdId)` (scrapea mediaId, CDN, audio tracks) → `pickBestAudio`
   (latino > español > inglés) → `pingSeek` → `waitForSegmentAt` (HEAD a `video-1.m4s`) →
   `_shakaLoad`. Seek: `tpipelineSeekTo` con debounce 300ms + mutex `_tReloading`. El tiempo
   real = `tpipelineOffset + video.currentTime` (Shaka resetea currentTime a 0 con `?t=`).
   Si falla → fallback transparente al transcode legacy.
4. **Transcode legacy** (`if rdId`): `fetchTranscode(rdId)` → DASH (`.mpd`) o HLS. Si DASH +
   `isDualLatFilename` → `probeSpanishDashTrack` (HEAD a pista española candidata).
   - **Si NO hay rdId pero sí `serverDashUrl`/`serverHlsUrl`/...** (ADR-004): se llama
     `onStreamReady({hasNativeSpanish:false})` y se reproduce el DASH/HLS/directo server-side
     (Shaka/hls.js) — termina aquí, sin caer a iframe.
4. **Reproducir**:
   - `hlsUrl` DASH → `loadDashViaShaka` (Shaka Player).
   - `hlsUrl` HLS → `loadHlsTranscoded` (hls.js).
   - `streamIsX265 && fallbackUrl` → `loadX265FallbackDirect`.
   - else → `loadDirectPlay`.

**"Audio malo"** (no directo): AC3/DTS/TrueHD/Atmos/DD+/FLAC/PCM/MP2, o MKV sin AAC
(con rdId). Regex en `streamSelector`.

**Límite de `/t/` (MEDIDO, no supuesto — 2026-08-02):** con `curl`, fuera del navegador,
6 segmentos (30 s de video) tardaron 43 s tanto en secuencial como EN PARALELO. Que ambos
den idéntico descarta cualquier causa de cliente (prefetch, buffer, conexión): es el techo
del servidor de RD para ese archivo. El mismo archivo por HTTP Range da 8,1 MB/s contra los
292 KB/s que necesita. Conclusión: si una copia se puede reproducir sin conversión, hay que
reproducirla así; no hay ajuste de cliente que compense a `/t/` generando bajo 1x.

**Cambio automático de copia (Plan B): ELIMINADO el 2026-08-02** a pedido explícito del
usuario. Una sola copia, la que elige el scoring, y no se cambia en el camino. Si no se
sostiene: recargas del monitor y, agotadas, cambio de reproductor. NO reintroducir.

**Watchdogs**: Shaka/HLS tienen timeout de ARRANQUE; si no arranca → toast + `onFallbackToNextSource`.

**Recuperación de stall a mitad de reproducción (DASH/Shaka):** además del watchdog de
arranque, un monitor (`setInterval` cada 2s) vigila que `currentTime` avance mientras está
en play. Causa típica: pausar varios minutos → las URLs de streaming de RD caducan → al
volver Shaka deja de avanzar ("se pega"). Si lleva ≥8s sin avanzar (y no está en pausa/seek/
fin/cambio-de-audio), **recarga el mismo DASH desde la posición actual** (`_shakaLoad` con
`startTime`) → reabre la sesión de RD. Tras `MAX_STALL_RECOVERIES` (3) intentos seguidos sin
éxito → `onFallbackToNextSource`. Respeta `isStale(myGen)`. Se limpia en `destroy()` y al
iniciar otra fuente. Constantes en `usePlayer.ts`: `STALL_CHECK_INTERVAL_MS`,
`STALL_RECOVER_MS`, `MAX_STALL_RECOVERIES`.

---

## 4. Carga de motores

- **Shaka Player** (`loadShakaIfNeeded`): script cdnjs 4.7.6 + `shaka.polyfill.installAll()`.
  Constructor correcto: `new window.shaka.Player()` (NO `new window.shaka()` — bug ya corregido).
- **hls.js** (`loadHlsIfNeeded`): cdnjs 1.4.12.
- Ambos con SRI (Subresource Integrity).

---

## 5. Audio español / cambio de pista

- `probeSpanishDashTrack(dashUrlBase)`: itera `SPANISH_TRACK_CANDIDATES`, construye la
  URL de la pista (`buildSpanishTrackUrl`) y hace **HEAD**; si responde ok →
  `hasNativeSpanish=true`, carga esa variante DASH (audio español).
- `switchAudioTrack(track)`: cambia pista en un DASH ya cargado (panel ⚙️).
  Solo disponible si `dashBaseUrl !== null` (`canSwitchAudio`).
- `isDualLatFilename(nombre)`: regex `dual|multi|lat|spa|español|castellano|latino`.

---

## 6. Subtítulos (`useSubtitles` + `onStreamReady`)

Disparo: `VideoPlayer.onStreamReady` — si `!hasNativeSpanish && selected.imdbId` →
`subtitles.fetchAndInject(...)`.

`fetchAndInject`:
1. Búsqueda OpenSubtitles por **release name** → fallback por **IMDB ID**.
2. `pickBestSubtitle` (scoring) + offset compartido (function `offset`).
3. Descarga del `.srt` vía function `subhash` (hasta 4 reintentos, valida ≥100 cues).
4. `inject(srt, offset)` → `parseSrt` → cues → overlay reactivo.

**Logs `[SUB]`** (console.warn) — restaurados 1:1, NO borrar.
Quirk: tras inyectar, status siempre `✅ ES | offset: 0s`.

### Caso "Weapons" (top DMCA/no cacheado) — RESUELTO (2026-06-09)
Síntoma: archivo multi-audio cuyo stream top (x265 AAC) está caído por DMCA
(`failed_infringement`) y sin `rdId` → no entraba a DASH → sin audio ni subtítulos
(caía a directo/iframe).
**Causa raíz:** no era detección de subtítulos, sino **selección de stream**. RD sí
tenía cacheada otra versión (x264 DDP Atmos), pero quedaba muy abajo en el ranking
(penalizada por audio incompatible) y el rescate original solo miraba el top-8 y solo
para audio malo.
**Fix:** en `streamSelector.ts → resolveActiveStream`, cuando el top no tiene `rdId`,
recorrer **TODO el pool** y usar la **mejor versión cacheada** (con rdId) → transcode
DASH → audio + subtítulos. Solo actúa cuando ya no había `rdId` (no afecta lo que andaba).
El error CORS `failed_infringement` que persiste en consola es ruido esperado.

> Sub-tema aún latente (no bloqueante): si un multi-audio va por **HEVC directo** (no
> DASH) con `isDualLatFilename=true`, `onStreamReady` recibe `hasNativeSpanish=true` y
> podría omitir subs. No se ha reproducido tras el fix anterior; reabrir solo si aparece.

---

## 7. Topbar / controles (PlayerView + VideoPlayer)

- **Auto-hide** (3.5s inactividad): oculta topbar + selector. Flecha "Volver" se oculta
  también en iframe (a pedido; en iframe el mousemove no burbujea, hay que mover a las
  franjas o Esc para reaparecer).
- **Controles** custom (`useNetflixControls`): play/seek/volumen(colapsable)/velocidad.
- Click en el `<video>` = play/pausa.
- Botones fullscreen reactivos (`:d` según `isFullscreen`, no `innerHTML`).

### Recuperación de stall/seek — 2026-06-13 (`usePlayer.ts`)
- Watchdog GENÉRICO en TODOS los caminos (DASH/HLS/directo): vigila que `currentTime` avance; si se
  traba recupera recargando desde la posición (función de recuperación por camino). Umbral más corto
  post-seek; detección de **seek lejano colgado** (`video.seeking` pegado); **backoff** por intento
  (no reiniciar el transcode de RD una y otra vez) antes de caer al iframe.
- **Selección por seekabilidad:** vuelan los archivos **MP4+H264** (play directo, seek por Range);
  los **solo-MKV/HEVC** van por transcode RD → seek lejano lagea (límite del backend de RD, no del player).
- En TV: `scoreStream(s, isTv)` prefiere H264/1080p/MP4 y penaliza HEVC/4K; el H264 juega directo
  también en TV. Ver ADR-007 y **ADR-008** (evolución a RD-mediaInfos).

### Fullscreen en TV (webOS) — 2026-06-13
- **Durante la reproducción** el player NO toca el Fullscreen API nativo en TV (`useFullscreen`
  gatea `requestFullscreen`/`exitFullscreen` por `isTvMode`): el overlay CSS `.player-page`
  (`fixed; inset:0; z-9999`) ya cubre toda la pantalla; tocar el fullscreen nativo solo togglea la
  barra del navegador webOS.
- **Al cerrar** (`PlayerView.closePlayer`, gesto de "Volver"): en TV se pide
  `document.documentElement.requestFullscreen()` → oculta de forma fiable la barra del navegador que
  webOS re-muestra tras el ciclo de reproducción. NO se hace `exitFullscreen` en TV (quedaría visible
  la barra). Para que el nav sobreviva a ese `:fullscreen`, `NavTV` es `position:sticky` (ver
  `components.md`). Pendiente de afinar; fix definitivo = empaquetar como PWA/.ipk.

---

## 8. Diagnóstico rápido

| Síntoma | Causa probable |
|---|---|
| Arranca con `buffer +0.0s` tras "continuar viendo" | Doble-resume: `tryHevcDirectPlay` ya posiciona y `PlayerView` re-seekeaba al mismo punto, descartando el buffer. **Corregido 2026-08-02** (guard de 5 s, log "ya arrancó en la posición guardada — sin re-seek"). |
| Se "pega" tras pausar varios minutos | URLs de streaming RD caducadas. **Mitigado**: el monitor de stall recarga el DASH desde la posición (ver §3). |
| Salta a iframe, `rdId null` | Token RD inválido/expirado, o archivo no cacheado en RD. |
| Video sin audio | Códec de audio no soportado directo + sin DASH (sin rdId). |
| Sin subtítulos | `hasNativeSpanish=true` (DASH o filename), o `imdbId` null, o functions no levantadas (`npm run dev` en vez de `netlify dev`). |
| Sin panel audio/subs | `onStreamReady` no se llamó (camino directo/x265 sin DASH no lo llama — comportamiento original). |
| `shaka is not a constructor` | usar `new shaka.Player()` (ya corregido). |
