# Real-Debrid + DASH + navegador: problema de SEEK + AUDIO (brief autocontenido para otra IA)

> ⚠️ **RESUELTO (2026-06-16):** El problema descrito abajo se solucionó al descubrir el pipeline privado `/t/` de RD. Ver `architecture/realdebrid-api.md` § "API PRIVADA — Pipeline `/t/`". Fase 5 cancelada.

> Documento autocontenido para pedir una segunda opinión técnica. Se puede pegar/pasar a
> cualquier IA aunque no tenga acceso al resto de los archivos del proyecto.

## Objetivo
App web (Vue 3, sin servidor propio de transcode, en Netlify) que reproduce pelis vía
Torrentio + Real-Debrid (RD). Necesito: reproducción que permita **adelantar/retroceder
libremente (incluso saltar 40-60 min)** Y con **audio en español/latino o inglés**.
NO tengo servidor de transcodificación propio (solo frontend + funciones Netlify de ~10s).

## Stack
- Torrentio (addon Stremio) devuelve ~60-110 "streams" por película, cada uno etiquetado
  `[RD+]` (cacheado en RD = reproduce ya) o `[RD download]` (NO cacheado).
- Real-Debrid: `/streaming/transcode/{id}` → { dash(.mpd), apple(hls .m3u8), liveMP4(.mp4),
  h264WebM(.webm) }, y `/streaming/mediaInfos/{id}` → pistas reales (audio/sub/codec).
- Reproductor: Shaka Player para DASH; HLS.js fallback; <video> nativo para Direct Play.
- Subtítulos: OpenSubtitles.

## El problema central
En títulos cuya ÚNICA versión cacheada es **MKV con audio AC3/DTS** (ej. "El corredor del
laberinto 3", "Jack Ryan Ghost War 2026", "La Momia 2026"), al adelantar lejos (ej. min 40)
la reproducción se queda cargando indefinidamente. Y en archivos pesados (6GB) se corta
incluso en reproducción normal.

## PRUEBAS REALIZADAS Y RESULTADOS REALES

### 1. DASH transcode de RD (lo que usa la app) — Shaka Player
- Reproduce, audio AAC OK (RD convierte AC3→AAC), seek CORTO funciona.
- Al saltar lejos (~1h): el segmento `video-N.m4s` queda **pending** (sin error, esperando).
  RD genera los segmentos ON-DEMAND/secuencial → el tramo lejano no existe aún → cuelga.
- El reproductor OFICIAL de RD sí seekea su propio transcode vía un mecanismo interno
  ("Seeking Time") que **NO está expuesto en la API pública**.

### 2. Archivo DIRECTO crudo (link /d/ de RD) en <video> simple
- `<video controls src="https://...download.real-debrid.com/d/XXXX/Pelicula.x264.AC3.mkv">`
- RESULTADO: **seek LIBRE perfecto** (30/40/50 min, adelante y atrás, sin colgarse).
- PERO **SIN AUDIO**: el audio es AC3 y Chrome NO lo decodifica (imagen sí, sonido no).
- Conclusión: el seek nunca fue el problema; el archivo directo es Range-seekable. El bloqueo
  es el audio AC3 no reproducible en navegador.

### 3. liveMP4 (MP4 progresivo de /streaming/transcode)
- Idea: si fuera MP4 progresivo con AAC, daría seek libre + audio.
- VERIFICADO con curl (MISMO id): `.mpd` (DASH) → **HTTP 200**; `.mp4` (liveMP4) → **HTTP 503**
  body `{"error":"transcoding_error"}`.
- Conclusión: RD **NO genera** el MP4 progresivo para estos archivos. liveMP4 descartado CON evidencia.

### 4. Pre-cacheo (forzar a RD a descargar una MP4/H264/AAC no cacheada)
- Existen versiones MP4/H264/AAC (YTS) pero `[RD download]` (no cacheadas).
- addMagnet/selectFiles → RD se queda en **0%** (sin seeders del lado de RD para títulos nuevos),
  deja torrents muertos en la cuenta. Descartado.

### 5. mediaInfos
- Devuelve audio/sub reales. La clave del audio (`spa1`,`eng1`,`lat1`) ES el token de la URL de transcode.
- Sirve para elegir el track latino y para detectar subs embebidos (cuando existen).

## LO QUE SÍ FUNCIONA
Títulos cuya versión cacheada `[RD+]` es **MP4 + H264 + AAC** (ej. Deadpool, El Padrino YIFY):
el navegador los reproduce DIRECTO (Range) → **seek libre + audio**, perfecto.

## CAUSA RAÍZ (reparto del cuello)
~85% Real-Debrid (transcode on-demand sin seek expuesto; no genera MP4 progresivo; no
transcodifica archivos pesados a tiempo real) · ~10% reproductor · ~5% scoring.

## TABLA RESUMEN
| Fuente RD                  | HTTP | Seek libre | Audio AC3→? |
|----------------------------|------|-----------|-------------|
| /d/ crudo (MKV)            | 200  | SÍ        | NO suena (AC3 en Chrome) |
| DASH .mpd (transcode)      | 200  | NO (on-demand) | SÍ (AAC) |
| liveMP4 .mp4 (transcode)   | 503  | —         | — (transcoding_error) |
| MP4/H264/AAC cacheado      | 200  | SÍ        | SÍ |

## SOLUCIÓN ACTUAL EN LA APP (vigente)
- Scoring: prioriza MP4/H264/AAC cacheado (Direct Play). Latino > inglés+subs > castellano.
  En transcode prefiere archivo liviano (RD lo sostiene).
- Audio: en transcode solo usa pista español si es LATINO; si no, inglés + subs.
- Recuperación de stall: retoma en la última posición (no reinicia a 0).

## PREGUNTA PARA LA OTRA IA
¿Existe ALGÚN camino SOLO-cliente (navegador, sin servidor de transcode propio) para obtener
**seek libre + audio** de un archivo MKV/AC3 vía Real-Debrid?
1. ¿Reproducir el MKV directo (seekable) y "agregarle" el audio AAC del transcode de RD como
   pista separada sincronizada (MSE/dual source)? ¿Viable/práctico?
2. ¿Forma de que RD entregue un MP4 progresivo con audio compatible que SÍ funcione (el liveMP4
   da transcoding_error)?
3. ¿Decodificación AC3 en cliente (wasm) del audio del MKV directo es realista?
4. Si no hay forma sin servidor: confirmar que la única solución es transcodear AC3→AAC en
   infraestructura propia (ffmpeg) y servir un MP4 con faststart (Range) — "Fase 5".
