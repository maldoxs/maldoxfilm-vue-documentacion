# Real-Debrid API — Referencia

> Referencia de la API REST de Real-Debrid v1.0. Cómo se usa en MaldoxFilm → `player.md` y `integrations.md`.
> Flujo server-side → `adr/ADR-004.md`.
> Documentación oficial completa (pego de la web de RD).

- **Base URL:** `https://api.real-debrid.com/rest/1.0/`
- **OAuth2 base:** `https://api.real-debrid.com/oauth/v2/`
- **Límite:** 250 requests/minuto. HTTP 429 si excede (cuenta dentro del límite).
- **Auth:** `Authorization: Bearer {token}` o `?auth_token={token}`

---

## Autenticación

### Header
```
Authorization: Bearer {token}
```

> ⚠️ Token privado NUNCA en cliente. En MaldoxFilm vive server-side (Netlify Function).
> Para apps opensource: client_id público `X245A4XAIBGVM` (scopes: unrestrict, torrents, downloads, user).

### OAuth2 flows
- **Web:** three-legged OAuth2 (`/auth` → code → `/token`)
- **Mobile:** device code (`/device/code` → user_code → poll `/token`)
- **OpenSource:** device code + `/device/credentials` → client_id/client_secret bound to user → luego mobile flow
- **Refresh:** POST `/token` con `grant_type=http://oauth.net/grant_type/device/1.0` + refresh_token

---

## Endpoints

### GET /disable_access_token
Deshabilita el token actual. 204.

### GET /time
Server time raw (`Y-m-d H:i:s`). Sin auth.

### GET /time/iso
Server time ISO (`Y-m-dTH:i:sO`). Sin auth.

### GET /user
Info del usuario: id, username, email, points, locale, avatar, type (premium/free), premium (segundos restantes), expiration.

### POST /unrestrict/check
Verifica si un link es descargable. Sin auth.
- `link*` + `password` opcional
- Returns: host, link, filename, filesize, supported (0/1)
- 503 si file unavailable

### POST /unrestrict/link
Desrestringe un link → URL directa de descarga.
- `link*` + `password` + `remote` (0/1)
- Returns: id, filename, mimeType, filesize, link, host, **chunks** (max parallel), crc, **download** (URL CDN), **streamable** (0/1)
- Para links múltiples (Youtube): type + alternative[]

### POST /unrestrict/folder
Desrestringe un folder link → array de URLs.

### PUT /unrestrict/containerFile
Decrypt container file (RSDF, CCF, CCF3, DLC) → array de URLs.

### POST /unrestrict/containerLink
Decrypt container file desde link HTTP → array de URLs.

### GET /traffic
Tráfico por hoster: left, bytes, links, limit, type (links/gigabytes/bytes), extra, reset (daily/weekly/monthly).

### GET /traffic/details
Detalle por día en un período (max 31 días). `start`/`end` en YYYY-MM-DD.

### GET /streaming/transcode/{id}
Manifiestos transcodificados. `id` de `/downloads` o `/unrestrict/link`.
- `apple`: { quality: m3u8 }
- `dash`: { quality: mpd }
- `liveMP4`: { quality: mp4 }
- `h264WebM`: { quality: webm }

### GET /streaming/mediaInfos/{id}
Información detallada de medios.
- filename, hoster, link, type (movie/show/audio), season, episode, year
- **duration**: float (segundos)
- bitrate, size
- details.video: { und1: { stream, lang, lang_iso, codec, colorspace, width, height } }
- details.audio: { und1: { stream, lang, lang_iso, codec, sampling, channels } }
- details.subtitles: [{ stream, lang, lang_iso, type (ASS/SRT) }]
- poster_path, audio_image, backdrop_path

### GET /downloads
Lista de descargas del usuario. Paginado (offset/page, limit ≤5000, default 100).
- id, filename, mimeType, filesize, link, host, chunks, download, generated
- type (calidad) solo si aplica

### DELETE /downloads/delete/{id}
Elimina un link de la lista de descargas. 204.

### GET /torrents
Lista de torrents. Paginado. Filtro `?filter=active`.
- id, filename, hash, bytes, host, split, progress (0-100)
- **status**: magnet_error, magnet_conversion, waiting_files_selection, queued, downloading, downloaded, error, virus, compressing, uploading, dead
- added, links[], ended, speed, seeders

### GET /torrents/info/{id}
Info completa de un torrent.
- Todo lo de arriba + original_filename, original_bytes
- **files[]**: { id, path (starting with /), bytes, selected (0/1) }

### GET /torrents/activeCount
{nb, limit} — torrents activos actuales vs máximo.

### GET /torrents/availableHosts
Hosters disponibles para subir torrent: { host, max_file_size }.

### PUT /torrents/addTorrent
Subir archivo .torrent. 201. Returns { id, uri }.

### POST /torrents/addMagnet
Añadir magnet link. 201. Returns { id, uri }.
- `magnet*` + `host` opcional

### POST /torrents/selectFiles/{id}
Selecciona archivos del torrent y arranca. 204.
- `files`: IDs separados por coma o "all"
- Para obtener IDs: `/torrents/info/{id}` primero
- 202 si action already done

### DELETE /torrents/delete/{id}
Elimina torrent. 204.

### GET /hosts
Hosters soportados. Sin auth. { domain: { id, name, image } }

### GET /hosts/status
Estado de hosters + competidores. Sin auth.
supported (0/1), status (up/down/unsupported), check_time, competitors_status.

### GET /hosts/regex
Regex de links soportados. Sin auth.

### GET /hosts/regexFolder
Regex de folder links soportados. Sin auth.

### GET /hosts/domains
Dominios soportados. Sin auth.

### GET /settings
Configuración del usuario.
- download_ports[], download_port
- locales{}, locale
- **streaming_qualities[]**, **streaming_quality**
- **mobile_streaming_quality**
- **streaming_languages{}**, **streaming_language_preference**
- **streaming_cast_audio_preference**

### POST /settings/update
Actualiza un setting. 204.
- `setting_name*`: download_port, locale, streaming_language_preference, streaming_quality, mobile_streaming_quality, streaming_cast_audio_preference
- `setting_value*`: valores posibles desde GET /settings

### POST /settings/convertPoints
Convierte puntos de fidelidad. 204.

### POST /settings/changePassword
Email de verificación para cambiar contraseña. 204.

### PUT /settings/avatarFile
Sube avatar. 204.

### DELETE /settings/avatarDelete
Resetea avatar a default. 204.

---

## Códigos de error numéricos

| code | Significado | Manejo en app |
|---|---|---|
| -1 | Internal error | reintento |
| 1 | Missing parameter | revisar request |
| 2 | Bad parameter value | revisar request |
| 5 | **Slow down** | **backoff + reintento** |
| 7 | Resource not found | fallback |
| 8 | **Bad token** | **token expirado → regenerar .env** |
| 9 | Permission denied | scope/token |
| 10 | Two-Factor auth needed | — |
| 11 | Two-Factor auth pending | — |
| 12 | Invalid login | — |
| 13 | Invalid password | — |
| 14 | Account locked | — |
| 16 | Unsupported hoster | — |
| 19 | Hoster temporarily unavailable | reintento |
| 21 | **Too many active downloads** | **limpiar torrents** |
| 22 | IP not allowed | — |
| 23 | Traffic exhausted | límite de cuenta |
| 24 | File unavailable | siguiente candidato |
| 25 | Service unavailable | reintento |
| 28 | File not allowed | — |
| 29 | Torrent too big | — |
| 30 | Torrent file invalid | — |
| 31 | Action already done | ignorar |
| 33 | **Torrent already active** | **reusar existente (buscar por hash)** |
| 34 | **Too many requests** (rate) | **backoff** |
| 35 | **Infringing file** (DMCA) | **saltar infoHash** |
| 36 | Fair Usage Limit | esperar |
| 37 | Disabled endpoint | — |

---

## Notas de implementación en la app

- **Camino cacheado:** `/downloads` → match → `/streaming/transcode/{rdId}` → DASH/Shaka.
- **Camino no-cacheado** (ADR-004, `rd-stream`): `addMagnet → info → selectFiles → poll(downloaded) → unrestrict/link → transcode`.
- **Subtítulos de RD — CERRADO (ADR-005):** DASH de RD NO expone pistas de subtítulos. Se mantiene OpenSubtitles.
- **DELETE /torrents/delete/{id}:** usado por `rd-cleanup` al cerrar/cambiar (ADR-006).
- **Duration** de `mediaInfos`: no se usa actualmente, podría servir para calcular seek boundaries.
- **Settings de streaming** (`streaming_quality`, `streaming_language_preference`): son para el player oficial de RD, NO afectan la API REST. No útiles para la app.

---

## Análisis: ¿sirve algo de esto para seek+audio?

Ningún endpoint de la API de RD permite:
- Pre-generar segmentos DASH (transcode bajo demanda, no cambiable)
- Solicitar un MP4 progresivo funcional (liveMP4 da 503 siempre)
- Extraer solo audio AAC del transcode como stream independiente seekable
- Cambiar el comportamiento del transcode (secuencial forzado por infra de RD)

**La API pública no expone nada que resuelva el far-seek en transcode-only.**

---

## 🕵️ API PRIVADA — Pipeline `/t/` (descubierto 2026-06-16)

Real-Debrid usa **internamente** un CDN con endpoints no documentados para servir video
a su propio reproductor web. Estos endpoints **sí resuelven el far-seek y el audio**.

### Base CDN

`https://{cdn}.real-debrid.com` — el CDN se obtiene de `mediaInfos` o del `<BaseURL>` del MPD.

### Endpoints

| # | Endpoint | Método | Auth | Descripción |
|---|---|---|---|---|
| 1 | `real-debrid.com/streaming-{rdId}?auth_token={token}` | GET | Query param | Página HTML de streaming. Contiene `setMediaId("{mediaId}")` |
| 2 | `app.real-debrid.com/rest/1.0/streaming/mediaInfos/{mediaId}` | GET | Bearer | JSON: `cdn`, `modelUrl` (hash), `duration`, `filename`, `audio[]` (id, lang, codec, channels) |
| 3 | `app.real-debrid.com/rest/1.0/streaming/ping/{mediaId}/play` | POST | Bearer | Activa el stream. Responde 200/204 |
| 4 | `app.real-debrid.com/rest/1.0/streaming/ping/{mediaId}/{seconds}` | POST | Bearer | Ordena generar segmentos desde `seconds`. Responde 200/204 |
| 5 | `{cdn}/t/{mediaId}/{hash}/{audio}/{subs}/{codec}/full.mpd?t={seconds}` | GET | Ninguna | MPD DASH con segmentos desde posición `seconds`. **Far-seek universal** |
| 6 | `{cdn}/t/{mediaId}/{hash}/{audio}/{subs}/{codec}/full.mp4` | GET | Ninguna | MP4 progresivo con HTTP Range (seek nativo del navegador) |

### Parámetros de URL

| Parámetro | Valores típicos | Descripción |
|---|---|---|
| `{audio}` | `eng1`, `spa1`, `fra1`... | ID de pista de audio (de `mediaInfos.audio[]`) |
| `{subs}` | `none` | Subtítulos (no usado aún) |
| `{codec}` | `aac` | Codec de audio (siempre AAC en este pipeline) |
| `{format}` | `mpd`, `mp4` | Formato de salida |
| `{quality}` | `full` | Calidad (siempre "full") |
| `?t={seconds}` | 0, 1850, 6420... | Posición de inicio en segundos |

### Flujo completo

```
rdId (de /downloads)
  → GET streaming-{rdId}?auth_token={token}
  → Extraer setMediaId("{mediaId}") del HTML (regex)
  → GET /streaming/mediaInfos/{mediaId}
  → cdn + modelUrl (hash) + audioTracks + duration
  → POST /streaming/ping/{mediaId}/play
  → POST /streaming/ping/{mediaId}/{seconds}  (para cada seek)
  → GET {cdn}/t/{mediaId}/{hash}/{audio}/none/aac/full.mpd?t={seconds}
  → Shaka Player reproduce
```

### Notas técnicas

- **`mediaId`** suele ser igual al `rdId`, pero no siempre. **Siempre verificar con el scrapeo.**
- **CDN mismatch**: `mediaInfos.cdn` puede diferir del CDN en los segmentos del MPD. El `<BaseURL>` del MPD es el que manda.
- **`full.mpd?t=X`**: Shaka pone `currentTime = 0` al cargar (timeline interna del MPD). Usar offset en el display.
- **`ping/{mediaId}/{seconds}`** no garantiza disponibilidad inmediata. Hacer polling con HEAD a `video-{N}.m4s` hasta 200.
- **`full.mp4`** funciona en algunos CDN pero no en todos. Usar como opción secundaria.
- **Shaka config**: `maxAttempts: 12, timeout: 15000, backoffFactor: 2` para este pipeline.
- **Fase 5 (servidor ffmpeg) cancelada**: este pipeline lo reemplaza.
