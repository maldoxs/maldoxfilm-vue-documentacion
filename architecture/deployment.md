# Deployment — MaldoxFilm (Netlify)

> Cómo correr en local y desplegar. Functions + Vite.

---

## 1. Local (DEV) — usar `netlify dev`

```bash
cd /Users/angelomaldonado/Desktop/maldoxfilm-vue
netlify dev          # abre en http://localhost:8888
```

`netlify dev`:
- Arranca Vite (puerto fijo 5173, `strictPort`) y lo proxea en **8888**.
- Sirve las **Netlify Functions** (`/.netlify/functions/*`).
- Inyecta las variables de `.env` (RD_TOKEN) en las functions.

> ⚠️ `npm run dev` (Vite puro, :5173) **NO** sirve functions → RD y subtítulos NO
> funcionan (token da 404). Usar solo para UI sin RD.

Al cambiar `.env` (token): **reiniciar `netlify dev`** (env vars solo se cargan al arrancar).

---

## 2. Config

`netlify.toml`:
```toml
[build]
  command = "npm run build"
  publish = "dist"
  functions = "netlify/functions"

[dev]
  framework = "vite"
  targetPort = 5173
  port = 8888
  # SIN `command`: netlify autodetecta Vite y lo arranca UNA vez
  # (si se pone command="npm run dev" además, Vite arranca 2 veces → "Port 5173 in use").
```

`vite.config.ts`: `server.port=5173`, `strictPort:true` (para que netlify dev siempre
lo encuentre).

`.gitignore`: ignora `.env`, `.env.*` (salvo `.env.example`), `.netlify`.

---

## 3. Functions

- `rd-token.js` — **ESM** (`export const handler`), porque `package.json` es
  `"type":"module"`. Lee `process.env.RD_TOKEN`.
- `rd-stream.js` — **ESM**. Resuelve no-cacheado server-side (ADR-004):
  `addMagnet→selectFiles→unrestrict→transcode` con varios `infoHash`, salta DMCA (error 35),
  reintenta 429/34/5. Lee `process.env.RD_TOKEN` (token NUNCA sale del servidor).
- `rd-cleanup.js` — **ESM**. Borra el torrent creado por `rd-stream`
  (`DELETE /torrents/delete/{id}`, ADR-006). Lee `process.env.RD_TOKEN`.
- `subhash.cjs`, `offset.cjs` — **CommonJS** (`.cjs`, `exports.handler`), copiadas
  verbatim del original. `offset.cjs` usa `@netlify/blobs` (dependencia instalada).

---

## 4. Producción (Netlify)

- Build: `npm run build` (`vue-tsc -b && vite build`) → `dist`.
- Functions: `netlify/functions`.
- **`RD_TOKEN`**: configurado en el panel de Netlify → Environment variables (NUNCA en
  el repo). Ya seteado por el usuario.
- Sitio: `maldoxfilm.netlify.app`.
- **Git conectado (desde 2026-06-12):** repo `github.com/maldoxs/maldoxfilm-vue`,
  branch `main`. **`git push origin main` → Netlify auto-build/deploy** (no hace falta
  `netlify deploy`). El repo de código vive en `/Users/angelomaldonado/Desktop/maldoxfilm-vue`.
- **SPA redirects:** `public/_redirects` con `/*  /index.html  200` (el router usa
  `createWebHistory`). Sin esto, entrar directo a una ruta o "volver" daba **404**.
  Se copia a `dist/_redirects` en el build. Las functions se resuelven antes que esta regla.

### Reglas de deploy (constraints)
- **Probar SIEMPRE en local (`netlify dev`) primero.**
- **No** desplegar a Netlify ni `git push` **sin permiso explícito** del usuario.
- Cada `netlify deploy --prod` DEBE incluir `--message`.
- Un cambio en producción a la vez.

---

## 5. Tests / type-check (pre-deploy)

```bash
npx vitest run        # 251 tests deben pasar
npx vue-tsc --noEmit  # EXIT=0
```
