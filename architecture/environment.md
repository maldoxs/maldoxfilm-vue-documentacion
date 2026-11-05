# Environment — MaldoxFilm

> Entornos, variables, URLs, puertos y dependencias externas.

---

## 1. Entornos

| Entorno | Cómo se levanta | Notas |
|---|---|---|
| **DEV (recomendado)** | `netlify dev` | Levanta Vite + **Netlify Functions** + inyecta `.env`. Único modo donde RD/subtítulos funcionan. Proxy en **:8888**. |
| **DEV (limitado)** | `npm run dev` | Vite puro en **:5173**. **NO** sirve las Functions → RD y subtítulos NO funcionan (token da 404). Útil solo para UI sin RD. |
| **PROD** | Netlify (`maldoxfilm.netlify.app`) | `npm run build` → `dist`. Functions en `netlify/functions`. Token en env vars del panel. |

> No hay QA/LAB formales; el ciclo es local (`netlify dev`) → prod (Netlify) con permiso.
> Comandos exactos y config (`netlify.toml`, build, reinicio) → única fuente en `deployment.md`.

---

## 2. Puertos

| Puerto | Servicio |
|---|---|
| **8888** | `netlify dev` (proxy principal — abrir aquí) |
| **5173** | Vite (fijo, `strictPort` en `vite.config.ts`; netlify dev lo proxea) |

---

## 3. Variables de entorno

| Variable | Dónde | Para qué |
|---|---|---|
| `RD_TOKEN` | `.env` (local, **gitignored**) / env vars de Netlify (prod) | Token Real-Debrid. Lo lee SOLO la Function `rd-token` (server-side). **Nunca** `VITE_*` ni en el bundle. |

- `.env` ejemplo (NO commitear): `RD_TOKEN=xxxxxxxx`
- `.env.example` (commiteable) documenta la variable sin valor.
- **Al cambiar el token: editar `.env` y REINICIAR `netlify dev`** (las env vars solo
  se cargan al arrancar).

> ⚠️ Política completa del token (manejo server-side, token comprometido, fases) →
> única fuente en `security.md`.

---

## 4. URLs y endpoints externos

| Servicio | Base / endpoint | Auth |
|---|---|---|
| **TMDB** | `https://api.themoviedb.org/3` | `api_key` pública (no secreta): `d95e6a7e2ead40e949fcdb81f1f26f0b` (en `bootstrap.ts`) |
| **TMDB imágenes** | `https://image.tmdb.org/t/p/{size}{path}` | — |
| **Torrentio** | `https://torrentio.strem.fun/...` | rdToken en la URL |
| **Real-Debrid API** | `https://api.real-debrid.com/rest/1.0` | `Authorization: Bearer {RD_TOKEN}` (server-side) |
| **RD CDN** | `https://{n}.stream.real-debrid.com/d/{rdId}/...` | — |
| **OpenSubtitles** | `https://api.opensubtitles.com/api/v1` | `Api-Key: BkJDwe4rThVSR3XRJ0F4Lhnn5loX6pYl` (en functions) |
| **Iframe fallback** | UnlimPlay (`unlimplay.com`), vidlink | — |
| **CDN libs** | hls.js (cdnjs 1.4.12), Shaka Player (cdnjs 4.7.6) | cargadas con SRI |

---

## 5. Netlify Functions (rutas internas)

| Function | Ruta | Runtime |
|---|---|---|
| `rd-token.js` | `/.netlify/functions/rd-token` | ESM (`export const handler`) |
| `subhash.cjs` | `/.netlify/functions/subhash?file_id=...` | CommonJS |
| `offset.cjs` | `/.netlify/functions/offset?infoHash=...&fileId=...` | CommonJS + `@netlify/blobs` |

Config en `netlify.toml`:
- `[build] command="npm run build", publish="dist", functions="netlify/functions"`
- `[dev] framework="vite", targetPort=5173, port=8888` (sin `command`, para no arrancar Vite dos veces).

---

## 6. Dependencias externas (npm)

`vue`, `vue-router`, `pinia`, `hls.js`, `@netlify/blobs`.
Dev: `vite`, `vitest`, `vue-tsc`, `typescript`, `@vitejs/plugin-vue`, `jsdom`, `@types/node`.

CDN en runtime (no npm): **hls.js** y **Shaka Player** se cargan dinámicamente desde
cdnjs cuando el reproductor los necesita.

---

## 7. Almacenamiento local (cliente)

| Key localStorage | Contenido |
|---|---|
| `sx_mylist` | Mi Lista |
| `sx_prog_v2` | Progreso "continuar viendo" |
| `sub_off_*` | Offsets de subtítulos guardados |

Netlify Blobs (servidor): store `subtitle-offsets` (crowdsourcing de offsets).
