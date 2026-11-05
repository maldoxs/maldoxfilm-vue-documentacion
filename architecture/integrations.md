# Integrations — MaldoxFilm (APIs externas)

> Servicios de terceros y cómo se consumen. Clients en `src/services/`.

---

## 1. TMDB (catálogo y metadata)

- Cliente: `services/tmdb.ts` (`get<T>(endpoint)`, `getImdbId(id,type)`).
- Base: `https://api.themoviedb.org/3`. API key **pública** (no secreta):
  `d95e6a7e2ead40e949fcdb81f1f26f0b` (en `bootstrap.ts`). Las keys TMDB son por diseño
  públicas (rate-limit por key) → OK que viajen al cliente.
- Imágenes: `https://image.tmdb.org/t/p/{size}{path}` (`buildPosterUrl` en `catalog.ts`).
- Endpoints usados: `/trending/...`, `/movie|tv/popular|top_rated|...`,
  `/discover/movie|tv?with_genres=...&sort_by=...`, `/movie|tv/{id}`, `/{type}/{id}/external_ids`.
- Consumo de listas: `useTmdbList` (una página) y `AllResultsView` (paginado infinito).

---

## 2. Torrentio (descubrir streams)

- Cliente: `services/torrentio.ts` (`fetchStreams({rdToken, imdbId, type, season, episode})`).
- Devuelve `streams[]` (cada uno con `name`, `title`, `url` de tipo `resolve/realdebrid/...`,
  `infoHash`, `behaviorHints.filename`).
- El `rdToken` viaja en la URL de Torrentio.

---

## 3. Real-Debrid (streaming HD)

- Cliente: `services/realdebrid.ts`.
  - `resolveProxyUrl(url)` → resuelve el proxy Torrentio a URL CDN real
    (`{n}.stream.real-debrid.com/d/{rdId}/...`).
  - `fetchDownloads()` → `GET /downloads?limit=500` (para matchear `rdId`).
  - `fetchTranscode(rdId)` → `GET /streaming/transcode/{id}` → DASH/HLS para reproducir.
- Auth: `Authorization: Bearer {RD_TOKEN}` — **server-side** (el token NO está en el
  bundle; ver `security.md`). En local los clients reciben el token resuelto por
  `bootstrap.ts` desde la function `rd-token`.
- El `rdId` está en la ruta del CDN (`/d/{rdId}/`) y es necesario para el transcode.
- **Camino no-cacheado (ADR-004):** si NO hay `rdId` en `/downloads` ni versión cacheada
  en el pool, se invoca la Function **`rd-stream`** (server-side): `addMagnet → selectFiles →
  unrestrict/link → streaming/transcode` con varios `infoHash`, saltando los DMCA (error 35),
  y devuelve DASH (multi-audio) con `liveMP4`/HLS de respaldo. Reemplaza la antigua URL
  `torrentio.../resolve/{TOKEN}/...` (que exponía el token y rompía Range).

> ⚠️ Si el token expira: `/downloads` falla → `rdId null` → no transcode → iframe.

> 📚 Referencia completa de la API de RD (endpoints, códigos de error, estados,
> rate limit, auth) → `realdebrid-api.md`. Doc oficial: https://api.real-debrid.com/

---

## 4. OpenSubtitles (subtítulos)

- Búsqueda (desde el cliente): `https://api.opensubtitles.com/api/v1/subtitles?...`
  con header `Api-Key` (en `useSubtitles.ts`). Por release name + fallback IMDB.
- Descarga del `.srt`: vía **Netlify Function `subhash`** (proxy server-side al
  `/download` de OpenSubtitles — evita CORS y mantiene la API key fuera del cliente).
  Devuelve `{link, remaining, file_name}`.
- Offsets crowdsourced: **Netlify Function `offset`** (Netlify Blobs, store
  `subtitle-offsets`): GET devuelve offset guardado, POST lo guarda.

---

## 5. Reproductores iframe (fallback)

- `playerSources.ts`: UnlimPlay (`unlimplay.com/play.php/embed/...`) y vidlink.
- Se usan cuando RD no entrega URL reproducible (`fallback-to-next-source`).

---

## 6. Librerías por CDN (runtime)

- **hls.js** 1.4.12 (cdnjs) — HLS transcode.
- **Shaka Player** 4.7.6 (cdnjs) — DASH transcode (multi-audio).
- Cargadas dinámicamente con SRI cuando el reproductor las necesita.

---

## 7. Netlify Functions (capa server-side propia)

| Function | Propósito | Archivo |
|---|---|---|
| `rd-token` | Entrega `RD_TOKEN` desde env (server-side) | `netlify/functions/rd-token.js` |
| `rd-stream` | Resuelve no-cacheado: addMagnet→selectFiles→unrestrict→transcode DASH (ADR-004) | `netlify/functions/rd-stream.js` |
| `rd-cleanup` | Borra el torrent creado por `rd-stream` al cerrar/cambiar (ADR-006) | `netlify/functions/rd-cleanup.js` |
| `subhash` | Proxy descarga OpenSubtitles + hash OS | `netlify/functions/subhash.cjs` |
| `offset` | Offsets de subs crowdsourced (Blobs) | `netlify/functions/offset.cjs` |

Solo funcionan con `netlify dev` (no `npm run dev`). Ver `deployment.md`.
