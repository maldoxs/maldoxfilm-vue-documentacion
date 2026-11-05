# Security — MaldoxFilm

> Manejo del RD_TOKEN y secretos. **Constraint crítico del proyecto.**

---

## 1. RD_TOKEN — server-side ONLY

El token de Real-Debrid da acceso total a la cuenta RD. **Reglas inviolables:**

1. **Nunca** en el bundle del cliente.
2. **Nunca** en `.env` con prefijo `VITE_` (Vite lo inlinearía en el JS público).
3. **Nunca** hardcodeado en código.
4. **Nunca** commiteado a git.

### Cómo se maneja
- En **local**: vive en `.env` (gitignored). Lo lee SOLO la Netlify Function
  `rd-token.js` (`process.env.RD_TOKEN`), que corre server-side.
- En **prod**: variable de entorno del panel de Netlify.
- El cliente obtiene el token en runtime con `fetchRdToken()` (`bootstrap.ts`), que
  hace `GET /.netlify/functions/rd-token`. Si la function no responde → token `''`
  (modo degradado seguro: RD inactivo, la app no se rompe).

### Token comprometido
- El token original `Z5EQ6…7HQA` (enmascarado) estaba
  **hardcodeado en el cliente del original → COMPROMETIDO/PÚBLICO**. Prohibido reusarlo.
- Si un token se rota, actualizar `.env` (local) o env var (prod) y reiniciar.

### Fase 1 vs Fase 2
- **Fase 1 (actual):** la function `rd-token` DEVUELVE el token al cliente en runtime
  (viaja por la red, visible en Network tab, pero NO en el bundle ni en git).
- **Fase 2 (ideal, futura):** la function actúa como **proxy completo** de RD/Torrentio,
  de modo que el navegador JAMÁS recibe el token. Documentado en `bootstrap.ts` como mejora.

---

## 2. OpenSubtitles API key

- Key (`BkJDwe4rThVSR3XRJ0F4Lhnn5loX6pYl`) se usa para BÚSQUEDA desde el cliente
  (bajo riesgo) y para DESCARGA desde la function `subhash` (server-side, evita CORS).

---

## 3. TMDB API key

- Pública por diseño (rate-limit por key, no secreta). OK que viaje al cliente.

---

## 4. Otras prácticas

- **Subtítulos como contenido no confiable**: los `.srt` vienen de OpenSubtitles. El
  overlay NO usa `v-html`; renderiza texto plano con `white-space:pre-line` (evita XSS).
- `parseSrt` limpia tags con regex, pero el render plano es la defensa real.
- Datos sensibles nunca en query params/URLs.

---

## 5. ⚠️ RIESGO PRINCIPAL + plan de Auth gate (PENDIENTE, decidido 2026-07-01)

**El riesgo #1 de MaldoxFilm NO es XSS/inyección — es que la app es PÚBLICA y usa la
cuenta PERSONAL de Real-Debrid del usuario.** Cualquiera que encuentre
`maldoxfilm.netlify.app` puede usar las Functions y consumir esa cuenta RD → posible
**baneo de la cuenta RD** o exposición a **DMCA**, además de costo/abuso.

**Decisión (audiencia = personal/familiar):** agregar un **Auth gate simple** es la medida
de mayor impacto (más que rate limiting o CSP para este caso). Con auth, solo entran
personas de confianza → el rate limiting deja de ser urgente.

### Plan de implementación (~0.5 día) — MISMO patrón que RD_TOKEN
1. Password/PIN vive en **env var de Netlify** (`APP_PASSWORD` o similar), NUNCA en el cliente.
2. Function `auth` valida el password → emite un **token firmado** (HMAC/JWT con un secreto server-side).
3. El cliente guarda el token (localStorage) y lo manda en cada request a las Functions.
4. `rd-token` / `rd-stream` / `rd-cleanup` / `subhash` **exigen** el token → rechazan si falta/inválido.
5. Pantalla de login antes de la app (o ruta `/login`); estado "no autenticado" → redirige.

### Otros items de seguridad (prioridad BAJA con auth puesto)
- 🟢 **Validación de URLs de streams** (2-4 hrs, defensa en profundidad): allowlist de hosts
  esperados (CDN de RD, Torrentio, embeds de anime) antes de usar una URL.
- ⬇️ **Rate limiting** en Functions: opcional con auth. Requiere store externo (Functions son
  stateless; Upstash Redis o Netlify Blobs —ya presente pero no ideal para contadores).
- ⬇️ **CSP headers**: frágil (puede romper reproductores iframe / imágenes TMDB) → después y con
  mucho testing. Riesgo XSS bajo con usuarios de confianza.
