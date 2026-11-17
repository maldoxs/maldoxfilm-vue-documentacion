# Project Memory — MaldoxFilm

> Estado VIGENTE del proyecto. No es bitácora histórica (eso va en `sessions/`).
> Última actualización: 2026-11-16 noche (validación de copias con mediaInfos de RD)

## 0.0 ⚡ Puntos de referencia (2026-11-16)

| Etiqueta | Commit | Estado |
|---|---|---|
| **`estable-2026-11-15-noche`** | **`63598bd`** | **Último punto estable verificado en uso.** |
| `validaciones-audio-2026-11-16` | `2bf87ee` | Desplegado, 485 tests verdes. El camino de respaldos **no está probado todavía en la TV**. |

Volver al punto estable (pierde todo lo del 16-11):

```
git reset --hard estable-2026-11-15-noche && git push origin main --force
```

### Qué hacía `63598bd` y lo que trae ese punto (2026-11-15)

- **Selección de copia:** un `.avi` no entra a la clase A (Cat's Eye); la copia en el
  idioma original suma +80 y una doblada a un idioma ajeno sale de la clase A
  (Frontier(s)); regla de selección escrita al principio de `streamSelector.ts`.
- **Ficha:** Dirección y Escritura.
- **Sincronía de subtítulos (`63598bd` en sí):** la medición corre dos métodos (por
  área y por arranques de voz) sobre los mismos tramos y solo aplica el ajuste si
  coinciden con menos de 2 s de diferencia; si no, avisa y ofrece "buscar más a fondo"
  (32 tramos). Antes se aplicaba un único método que se equivocaba por 35 s en algunas.

### Qué hace `2bf87ee` y lo que trae ese punto (2026-11-16)

Caso *Dreamcatcher*: sonó en checo, luego mudo. La copia elegida (MP4 en inglés) no
abría a tiempo y el respaldo se elegía por el nombre.

- Toda copia, elegida o respaldo, se valida con `mediaInfos` de RD: la pista que va a
  sonar se decodifica en el equipo; video que se muestra; pista preferida primera o
  cambiable; alguna pista en latino, español, original o inglés.
- Si la elegida falla se prueban hasta 6 copias cacheadas verificadas, en paralelo.
- En TV, si ninguna sirve, siguiente reproductor en vez de convertir (decisión del
  usuario). En escritorio, conversión como siempre.
- En TV, 25 s para abrir una copia que RD ya confirmó (antes 10 s).
- Los torrents que agrega la verificación se borran al cerrar (ADR-006).
- Checo, húngaro, sueco, hindi y árabe cuentan como doblaje ajeno.

Detalle completo en [`sessions/2026-11-16.md`](sessions/2026-11-16.md).

## 0.0-hist00 ⚡ Estado anterior (2026-10-18 noche)

**HEAD `a8ce410`.** 418 tests verdes, `vue-tsc -b` OK, build OK, desplegado y
verificado en producción. Punto estable para volver atrás:
`estable-2026-10-17-noche` → `1970512`.

### Dónde está el cuadro completo

El flujo entero, de TMDB a la pantalla, con los motivos de cada decisión, está
en [`architecture/flujo-reproduccion.md`](architecture/flujo-reproduccion.md).
Es el documento para leer antes de tocar selección, reproducción o subtítulos.

### Subtítulos: la medición automática

- El botón "ajustar al audio" escucha ocho tramos de un minuto. Si el margen de
  confianza queda al borde del corte (1,25), hace una **segunda pasada con
  dieciséis**. Medido con *Hope*: 1,245 con ocho, 1,30 con dieciséis, y el número
  bueno estaba en las dos.
- **El idioma no influye** en que la medición funcione; lo que influye es cuánta
  voz tiene la película. Márgenes medidos: Frontiers (francesa) 1,52, The Dark
  and the Wicked 1,47, Ghostland 1,41, Hope (coreana, muy silenciosa) 1,245.

### Reproducción: tolerancia a la pausa

- **El enlace de Real-Debrid ya no se corta en las pausas.** Mientras la película
  está pausada se lo toca cada dos minutos con un `HEAD` en modo `no-cors`.
  Verificado en la TV: **pausa de 24 minutos y siguió directo**, con el buffer
  intacto y sin un solo aviso.
- **Si igual se corta, reconecta sola**: vuelve a resolver el stream y retoma
  tres segundos antes, con el aviso "Reconectando". Dos intentos como máximo.
  Verificado: corte real en *Tenzing*, retomó en seis segundos.
- Ojo con esto si se toca el manejador de `error` del `<video>`: tiene que seguir
  distinguiendo el intento INICIAL (descartar la copia) de una caída en plena
  reproducción (reconectar). Confundirlos es lo que dejaba la pantalla negra en
  00:00.
- **Al terminar la película quedan los controles a la vista**, si no la barra ya
  estaba escondida y no se veía la flecha de volver.
- La memoria de la pestaña se registra en cada evento del log. Medida en varias
  sesiones largas: **10 de 273 MB**. La TV no se queda sin memoria.

### Subtítulos

- **Se buscan por la HUELLA del archivo, no por su nombre.** La function
  `moviehash` calcula la huella OSDb (tamaño + primeros y últimos 64 KB) y
  OpenSubtitles devuelve los subtítulos subidos para esa copia exacta, que por
  definición están sincronizados. Un candidato con `moviehash_match` suma 40
  puntos: le gana a cualquier indicio de texto.
- La consulta por huella va **sin filtro de idioma** a propósito: aunque no haya
  subtítulo en español para el archivo, el que exista en cualquier idioma revela
  el **nombre real del release**, que suele no tener nada que ver con el nombre
  del torrent. Con ese nombre se busca en español y salen las pistas de calidad.
- Esa function **no puede usar el enlace de Torrentio**: Torrentio responde 403 a
  los servidores (solo atiende navegadores). Le pide el enlace directo a
  Real-Debrid con el identificador de la DESCARGA (no el de un torrent, que da
  404).
- **Botón "ajustar al audio"** en el panel de Sincronización: mide el desfase
  real contra el audio de la película y lo corrige solo, en alrededor de un
  minuto. Validado en tres películas, dos corridas y una sana (ver
  `sessions/2026-10-18.md`, que incluye la lista de enfoques que NO funcionan,
  para no reintentarlos).
- **El ajuste compartido entre reproducciones nunca había funcionado.** La
  function `offset` estaba en el formato viejo y Netlify la corría sin acceso al
  almacenamiento; fallaba en silencio desde siempre. Reescrita.

### Reproductores

- **Se acabó el ida y vuelta RD → UnlimPlay → RD.** Cada fuente se intenta una
  vez por título, la cadena avanza y, agotadas las tres, sale el cartel
  "Película no encontrada" con un botón que vuelve al detalle. Lo que lo
  provocaba era que la página embebida recargaba la app entera cada seis
  segundos; no se puede bloquear (vidlink se niega a cargar con `sandbox`), así
  que lo que se corta es el bucle: las fuentes descartadas se recuerdan tres
  minutos en `sessionStorage`.
- Un códec que el navegador no muestra (mpeg4/DivX/Xvid y compañía) ya no se
  intenta: se cambia de reproductor.

### TV

- **Ya no se detecta como celular.** El navegador de los LG dice `Web0S` con un
  cero, variante que faltaba en la lista de marcas. Además el modo `tv` queda
  anotado en `localStorage` y no se degrada en cargas posteriores.
- El identificador del navegador de esa TV **cambia entre cargas**: `NetCast /
  Colt` en el primer arranque, `Windows / Whale` después de una recarga. Es el
  mismo televisor.


## 0.0-hist0 ⚡ Estado anterior (2026-07-13 noche)

- **HEAD `9e856cd`.** 305 tests verdes, `vue-tsc -b` OK, build OK. App EN VIVO, todo
  pusheado y desplegado. Ver `sessions/2026-07-13.md` (Partes 8-20) para el detalle
  completo de esta sesión (la más larga hasta ahora).
- **✅ RESUELTO Y CONFIRMADO — audio idéntico en todos los dispositivos** (bug real
  "Parásitos": sonaba latino en desktop, coreano en TV). Causa: la ronda "upgrade a
  fluido" de `resolveActiveStream` (streamSelector.ts) podía cambiar el stream elegido
  por CUALQUIER versión Direct Play cacheada, sin mirar el idioma — dependía de qué
  cacheara Torrentio en cada request. Fix: `audioLangRank` — el upgrade nunca puede
  bajar el idioma (latino→coreano). Confirmado con `rdId` idéntico en TV y desktop.
- **✅ RESUELTO Y CONFIRMADO — "El Padrino" con audio funcionando** (incluso en
  LATINO). Causa real (NO era límite de RD, como se pensó al principio — el usuario
  aportó evidencia del reproductor oficial de RD que lo desmintió): el archivo ya
  estaba cacheado en la cuenta, agregado MANUALMENTE vía "Torrents to Direct
  Download", con un hash/nombre que Torrentio NUNCA ofrece como stream → ninguna
  ronda de búsqueda por infoHash lo encontraba. Fix: `findCachedByTitleYear` —
  rescate de último recurso que busca en TODOS los downloads de la cuenta por
  TÍTULO+AÑO (no por hash). Aplica a CUALQUIER película, no solo esta.
- **✅ Fase 2 implementada**: `rd-stream` por fases (`rd-stream-start/-status/-finish`)
  con polling desde el cliente (sin techo de 9s de Netlify) — para títulos sin `rdId`
  cuyas mejores versiones estén bloqueadas por DMCA, prueba más candidatos salteando
  bloqueos. `rd-stream.js` monolítico queda como fallback (nunca peor que antes).
- **✅ Otros fixes de la sesión** (ADR-009, todos en `main`): cues del TextTrack
  nativo en `/t/` corregidos con offset (desface de subtítulo en móvil/iOS), piso de
  confianza + verificación IMDB en selección de subtítulos (evita subs de otra
  película), `mediaInfos` antes de Direct Play cuando hay `rdId` (audio real, no por
  nombre). **Descartado**: probe de audio post-arranque (`webkitAudioDecodedByteCount`)
  — implementado y REVERTIDO por falso positivo (cambiaba de reproductor en títulos
  que sí andaban bien). Lección: no confiar en esa API para tomar acciones destructivas.
- **✅ UX nueva**: "Ver desde el inicio" (estilo Netflix) — botón en `DetailView`
  (solo si hay progreso guardado, navega con `?start=1`) y DOS botones dentro del
  reproductor (junto a la flecha "Volver" arriba, y en la barra de controles abajo),
  ambos reinician a 0 sin salir del player. El progreso persistido NO se borra.
- **Panel de diagnóstico en pantalla (🐞)**: se agregó esta sesión para depurar en la
  TV sin consola, cumplió su propósito, y se RETIRÓ del código.
- **Abierto (sin confirmar)**: si el desface de subtítulo reportado en TV al
  principio de la sesión sigue apareciendo, o si ya quedó cubierto por el fix de
  cues `/t/` (ese fix corrige el camino móvil; TV usa el overlay DOM, que en teoría
  ya compensaba el offset desde antes). Pendiente de que el usuario lo confirme en
  una próxima prueba.
- **Sin tocar en esta sesión** (por regla del proyecto): scoring de `streamSelector`
  (pesos +400/+200), timing de buffer/heartbeat — la barra gris sigue sólida, sin
  regresión.

---

## 0.0-hist ⚡ Estado anterior (2026-07-13, madrugada — rollback de emergencia)

- **HEAD `a8cc1e7`.** Un rollback de emergencia (crisis de reproducción con timing de
  buffer) llegó hasta `04f1d73` (27-jun) — MUY profundo, borró ~2 semanas de trabajo
  real (rediseño mobile, compat TV, iOS, heartbeat). Se recuperó moviendo `main` a
  `de90173` (13-jul, ya trae todo ese trabajo + los experimentos malos ya revertidos
  dentro de esa historia) y re-aplicando encima 2 fixes nuevos (`SEEK_FREEZE_NEAR_MS`
  12s→45s, `hasNativeSpanish` vía `hasLatinoTag`). Ver `sessions/2026-07-13.md` Parte
  5-7 para el detalle y las lecciones (no tunear buffer/arranque especulativamente;
  medir cuánto trabajo hay entre medio antes de un `reset --hard` profundo).

---

## 0.0-hist2 ⚡ Estado anterior (2026-07-01)

- **Capa de compatibilidad TVs viejas (Chromium 38 / LG webOS 3, p.ej. UH8500) DESPLEGADA**
  (HEAD `4eaca8e`). 3 pasos: plugin-legacy (JS→ES5), postcss aplana `var(--)`, autoprefixer +
  fallbacks grid→flex/aspect-ratio dentro de `@supports not()`. **Verificado moderno intacto +
  usuario confirmó móvil/desktop/TV moderna OK.** Pendiente: validar UH8500 (2026-07-02). Tags
  de restauración: `antes-estilo-teles-antiguas` (revertir todo) y `compat-legacy-ok`. Detalle
  en `sessions/2026-07-01.md`. **NO tocar el navegador moderno: todos los fallbacks viven en
  `@supports not(...)` — el moderno los ignora.**
- **Roadmap acordado**: 1) validar UH8500, 2) bug anime móvil (iOS, ABIERTO), 3) PWA
  (descargable), 4) seguridad (auth gate #1 — ver `architecture/security.md §5`).
- **Seguridad**: audiencia = personal/familiar. Riesgo #1 = app pública usando RD PERSONAL →
  **auth gate (password/PIN)** es prioridad (protege la cuenta RD). Rate limiting/CSP → opcionales.

---

## 0. ⚡ Reproductor — verdad operativa (lo más importante de saber)

- **Pipeline `/t/` (RD CDN)** es el MÉTODO PRINCIPAL y CANÓNICO de streaming. Usa la API privada de RD:
  - `full.mpd?t={seconds}` → DASH con segmentos generados **en cualquier posición** → **far-seek UNIVERSAL** ✅
  - `full.mp4` → progresivo con HTTP Range → seek perfecto.
  - Proporciona audio **AAC** siempre → AC3/DTS ya no es problema.
  - Obtén `mediaId` vía bootstrap (`/streaming-{rdId}?auth_token={token}`, extraer `setMediaId("{mediaId}")`).
  - Actívalo con `ping/{mediaId}/play` y genera posición con `ping/{mediaId}/{seconds}`.
  - Servicios: `rd-t-pipeline.ts` (cliente + URLs + waitForSegmentAt + pickBestAudio), `netlify/functions/rd-tpipeline.js` (server-side: resolve/seek/mpd, token nunca al cliente).
  - **NO es un tier del scoring** — es un upgrade transparente en `usePlayer.ts` (`tryTpipeline`). Se intenta SIEMPRE que hay `rdId`; si falla → fallback a transcode legacy sin corte.
  - `pickBestAudio`: latino > español > inglés > primera disponible.
  - Seek: debounce 300ms + mutex `_tReloading` + freeze-frame (canvas drawImage) + spinner.
  - `waitForSegmentAt`: chequea `video-1.m4s` (relativo al `?t=`), NO el absoluto.
- **Selección** (`streamSelector.ts scoreStream`) por tiers: **P1 cacheado `[RD+]` = +400** → P2 H264 → P3 AAC → P4 tamaño → P5 1080p → P6 idioma → P7 MP4. ⚠️ **2026-06-27: P1 subió de +200 a +400.** Razón: con +200 EMPATABA con el bonus latino (+200) → un inglés-cacheado (`/t/` fluido) y un latino-NO-cacheado (server-side lento) quedaban parejos, y como Torrentio devuelve la lista distinta en cada request, el desempate caía distinto en TV vs desktop → MISMO título daba `/t/` en uno y server-side en otro. Con +400 el cacheado SIEMPRE gana sobre no-cacheado → siempre `/t/` fluido en TODAS las pantallas (un latino CACHEADO +600 sigue ganando). **NO bajar ese +400.** Trade-off: latino no-cacheado → inglés+subs (fluido) en vez de latino server-side (lento).
- **NO hay lógica de reproducción por dispositivo.** `usePlayer.loadRdSource` pasa `isTv=false` a `getStream` y `hevcOk` se decide igual que desktop. TV = desktop en TODO el flujo. (La "preferencia latino server-side" de rdStream.ts fue DESHABILITADA — degradaba `/t/` a server-side lento.)
- **Transcode DASH viejo** (`/streaming/transcode/{id}/dash.full`) ahora es **legacy/fallback** — solo se usa si no hay `mediaId`. Su far-seek limitado ya no importa.
- **Direct Play** (MP4/H264/AAC cacheado) = HTTP Range = seek perfecto, pero el `/t/` pipeline lo iguala (y además soporta cualquier codec con re-encode de RD). Ambos coexisten.
- **Red anti-congelamiento** (`usePlayer`): como el far-seek ya funciona, el riesgo de congelamiento por seek lejano DESAPARECIÓ en pipeline `/t/`. Solo aplica si cae a transcode legacy.
- **HÍBRIDO idioma/seek:** en `/t/` pipeline el audio se elige por ID real (`mediaInfos`) y el seek funciona → ya no hay trade-off. Se prefiere LATINO; si no → inglés+subs (NUNCA castellano España).
- **Subtítulos** (`scoreSubtitle`): prefiere HUMANO sobre AI-translated (−8). Offset manual persiste (safeStorage). OpenSubtitles sigue siendo la fuente.
- ❌ **Fase 5 (servidor ffmpeg propio) — CANCELLED:** el pipeline `/t/` de RD ya provee seek universal + AAC para cualquier archivo. No necesita servidor extra.
- ❌ **liveMP4 DESCARTADO:** `/streaming/transcode` devuelve 503 en `.mp4`.
- ❌ **Pre-cacheo DESCARTADO:** RD no descarga títulos nuevos no-cacheados. El pipeline `/t/` funciona con lo que ya está cacheado (el `mediaId` se obtiene igual).
- Política de audio: **latino/español preferido**; si no, **inglés + subtítulos español**.

### 🧪 Pipeline `/t/` — aprendizajes técnicos (demo 2026-06-16, portado 2026-06-21)

- **Offset `tpipelineOffset`**: `full.mpd?t=X` hace que Shaka ponga `currentTime = 0`. Tiempo real = `tpipelineOffset + video.currentTime`. La barra de controles usa `timeOverride`/`durationOverride`/`seekOverride` en `useNetflixControls`.
- **`waitForSegmentAt`**: con `?t=X` el primer segmento es `video-1.m4s` (relativo), NO `video-{X/5}.m4s` (absoluto). Chequear el absoluto daba timeout siempre → 8s de demora. Corregido 2026-06-21.
- **Debounce + mutex**: sin esto, arrastrar la barra lanza múltiples `_shakaLoad` → colisión (Shaka error). Debounce 300ms + `_tReloading` mutex. Corregido 2026-06-21.
- **Freeze frame**: canvas.drawImage del último cuadro + spinner azul centrado. Evita pantalla negra.
- **Corte de audio en seek**: `video.pause()` + `video.muted = true` antes de recargar Shaka. Restaurar después.
- **CDN mismatch**: la function `rd-tpipeline.js` ya resuelve el CDN real del MPD (puede diferir de `mediaInfos.host`).

---

## 1. Estado actual

Migración de **MaldoxFilm** de vanilla JS (`Streamix-local/assets/index.html`,
~9.600 líneas) a **Vue 3 + TypeScript + Vite**. La app es un catálogo de
películas/series (datos de TMDB) con un reproductor que usa **Real-Debrid (RD)**
para streaming HD y cae a reproductores iframe (UnlimPlay/vidlink) como respaldo.

- **Repositorio código:** `/Users/angelomaldonado/Desktop/maldoxfilm-vue/`
- **Original (referencia 1:1):** `/Users/angelomaldonado/Desktop/Streamix-local/assets/index.html`
- **Deploy:** Netlify (`maldoxfilm.netlify.app`) — **EN VIVO**, git conectado a GitHub
  (`github.com/maldoxs/maldoxfilm-vue`, branch `main`). Push a main → auto-deploy. `RD_TOKEN`
  configurado en env vars de Netlify (por el usuario). `public/_redirects` (`/* /index.html 200`)
  para SPA (rutas/volver ya no dan 404).
- **Tests:** 265 pasando, `vue-tsc --noEmit` limpio. HEAD `bac42f4` (live `index-CR-XGCaT.js`).

### Funciona hoy (verificado en vivo en TV + local)
- Reproducción RD (El Padrino, Jack Ryan Dual-Lat, Parásitos, The Thing, Weapons) con audio + subtítulos.
- **Capas de resolución RD:** cacheado (rdId) → rescate del pool (versión ya cacheada) →
  **server-side `rd-stream`** (addMagnet→…→DASH, ADR-004) para no-cacheado, sin Torrentio.
- **Guard anti-archivo-basura** (caso "Scary Movie"): descarta samples/test por nombre y por
  tamaño (<50MB); si el stream final es basura, anula la URL → cae a rd-stream/iframe, nunca reproduce basura.
- Detección/cambio de pista de audio (DASH multi-audio) e inyección de subtítulos ES (OpenSubtitles, ADR-005).
- Inicio + Películas/Series/Anime + "Ver todo" (grilla) + Mi Lista + Buscador.
- **Modo TV (LG/Magic remote) — sólido:** player a pantalla completa; nav TV con buscador/fullscreen
  a la derecha (logo Oswald, activo subrayado); buscador TV con teclado (Enter cierra teclado);
  tamaños de texto para sofá; controles de hover (flechas/✕) siempre visibles; **navegación
  espacial D-pad** por carruseles (←/→ card por card, ↓/↑ entre categorías, ↑ vuelve al nav activo);
  **foco sigue al puntero** (donde apuntás manda).
- **Menú de géneros estilo Netflix (TV)** (`TvGenreBar.vue`): activo anclado a la izquierda con
  borde blanco; se desliza; **scroll INFINITO** (lista triplicada, sin gap negro); filtro al
  asentarse (~350ms). Solo TV; escritorio/móvil mantienen la barra de pills.
- **Imágenes `loading="eager"`** (sin el efecto gris→foto al scrollear) y **nav TV fondo negro sólido**.
- **Optimización de memoria** (buffer Shaka recortado en TV/móvil). *(El `content-visibility` se
  quitó a pedido del usuario: no le gustaba el efecto de imágenes apareciendo al scrollear.)*
- **Recuperación de stall en reproducción** (DASH): si la peli se traba tras pausar, recarga sola desde la posición.
- **Limpieza de torrents RD** al cerrar/cambiar (ADR-006, function `rd-cleanup`).
- **Cierre limpio del reproductor:** al volver, se pausa/vacía el `<video>` y el iframe → no sigue
  el audio ni queda "espacio fantasma" (commit 5ba60e4).
- **Sin "línea azul" en TV:** se corrigió el bug de `:global(html.tv-mode) .x:focus` en scoped CSS
  que aplicaba un outline a todo el `<html>` (commit d74e3be). Reglas de foco tv-mode ahora en
  `style.css` global con `:focus-visible`.

### Sesión 2026-06-13 — TV/webOS (confirmado por el usuario)
- **Sin "mini footer" azul al pie:** era el SCROLLBAR horizontal del `<body>` (el player `width:100vw`
  desbordaba el gutter). Restaurado `body{overflow-x:hidden}` (perdido vs original) + scrollbar
  horizontal oculto + hover del thumb gris (no azul). (commit ae2d439)
- **Barra del navegador webOS ya no tapa el nav tras reproducir:** al "Volver" se pide
  `requestFullscreen` (oculta el chrome) y el **NavTV es `position:sticky`** (NO `fixed`) para
  sobrevivir al `:fullscreen` de webkit (donde `fixed` deja de pegarse al scrollear). (commit 6449a4b)
  ⚠️ Funciona pero "falta afinar"; el fix definitivo es empaquetar como PWA/.ipk (sin chrome).
- **Sin tarjetas grises (skeleton) al scrollear/volver:** skeleton transparente en TV + **cache
  stale-while-revalidate** en `useTmdbList` (memCache de sesión + `safeStorage`/localStorage TTL 6h).
  Las listas ya vistas se pintan al instante. (commit 6939a9c)
- **El nav mantiene la sección activa** en ficha/player/ver-todo (`lastSection` en `App.vue`, ya no
  salta a "Inicio") y el botón **"Volver" de la ficha** regresa a la sección+género previos
  (`router.back()`, el género viaja en `?genero=`), con fallback a Inicio si no hay historial.
  (commits 18295cb, 8cf3239)
- **Reproductor — fluidez al hacer seek:** recuperación de stall/seek UNIVERSAL (DASH/HLS/directo) +
  seek lejano colgado + backoff; en TV preferir H264/1080p/MP4 y HEVC al transcode (Deadpool se ve);
  H264 juega directo también en TV (seek por Range). Vuelan los títulos con MP4-H264; los solo-MKV/HEVC
  van por transcode RD y el seek lejano aún lagea (límite de RD). Tag `estable-pre-scoring-mp4`.

### ✅ Pipeline `/t/` — Demo validada (2026-06-16, `PRUEBAS/`)
Demo completa funcional: servidor Node (`server.mjs`) + frontend vanilla (`public/index.html`).
Far-seek universal confirmado. Seek a 1s (~1s), 30 min (~2s cacheado, ~10s primera vez),
50 min (~10s). Freeze frame, loader, corte de audio, barra custom con offset funcionando.
**Backup en `PRUEBAS_backup_20260616_2257/`.**
**Siguiente: adaptar aprendizajes al proyecto principal (`maldoxfilm-vue`).**

### Pendiente / a evaluar
- Secciones del Inicio que faltan vs producción: **Anime** y **Deportes (EN VIVO)**.
- Lógica de **subtítulos para multi-audio sin DASH-confirmado** (caso "Weapons"). Ver Riesgos.
- Badge contador "VER TODO 10K" en carruseles (decorativo, no implementado).
- **Afinar pipeline `/t/`**: elegir CDN correcta (`mediaInfos.cdn` vs la que usa el MPD), manejar errores 403, detectar `mediaId` inválido, auto-switch a transcode legacy.
- **Probar pipeline `/t/` en TV** (LG webOS) — puede tener limitaciones de DASH nativo.

---

## 2. Decisiones vigentes (constraints permanentes)

1. **Fidelidad 1:1 con el original**, incluidos quirks cosméticos (p.ej. el
   subtítulo siempre muestra `✅ ES | offset: 0s` tras inyectar, sin importar el offset real).
2. **RD_TOKEN es server-side ONLY.** Nunca en el bundle del cliente, nunca en git.
   Se sirve vía Netlify Function `rd-token` leyendo `process.env.RD_TOKEN` (de `.env`
   gitignored en local; de env vars del panel en prod). Ver `architecture/security.md`.
   - El token original (`Z5EQ6B2BAS6...`) está **COMPROMETIDO/PÚBLICO** — prohibido reusarlo.
3. **Probar en local primero** (`netlify dev`, puerto 8888). NO subir a Netlify/git
   sin permiso explícito del usuario. Cada deploy Netlify DEBE incluir `--message`.
4. **Responder en español** al usuario describiendo lo que se hace.
5. **No usar tools de preview/browser** para probar el reproductor — el usuario lo
   prueba en local y reporta logs.
6. Cada cambio en producción: **uno a la vez**.

---

## 3. Riesgos actuales

| Riesgo | Detalle | Mitigación |
|---|---|---|
| **RD_TOKEN expira** | Si el token de RD caduca, `/downloads` falla → `rdId null` → no transcode → cae a iframe. Síntoma: "Sin match en downloads — rdId null" y salto a UnlimPlay. **Estado: RESUELTO el 2026-06-09** — token nuevo en `.env`, verificado con `[RD] Match → id: 66UKOAEBEUWNG` (reproduce vía DASH). Riesgo recurrente (los tokens caducan). | Generar token nuevo en RD, actualizar `.env`, **reiniciar `netlify dev`**. NO es bug de código. |
| **Streams DMCA / no cacheados (top sin rdId)** | El stream mejor puntuado puede estar caído por DMCA (`failed_infringement`) o no cacheado → `rdId null` → antes caía al iframe (sin audio/subs). Caso "Weapons". | **RESUELTO (2026-06-09)**: el rescate en `resolveActiveStream` ahora recorre TODO el pool y usa la mejor versión cacheada (con rdId) → transcode DASH → audio + subs. Reproduce con audio inglés + subtítulos ES. |
| **HEVC/MKV no reproducible directo** | Códecs no soportados por el navegador requieren pipeline `/t/` o DASH. **RESUELTO** por `/t/` (re-empaqueta a DASH con AAC + seek). | Pipeline `/t/` es el camino; si no hay `mediaId` → transcode legacy → iframe. |
| **Functions locales** | `subhash`/`offset`/`rd-token`/`rd-stream` solo corren con `netlify dev` (NO `npm run dev`). | Documentado; usar siempre `netlify dev`. |
| **Acumulación de torrents en RD** | `rd-stream` hace `addMagnet`. | **RESUELTO (ADR-006)**: function `rd-cleanup` borra el torrent al cerrar/cambiar. (Sub-tarea: los `not_cached` que quedan descargando no se limpian.) |
| **`rd-stream` sin prueba end-to-end** | Función endurecida y cadena cliente coherente, pero el `ready:true` real aún NO se observó en vivo. | Probar con un título sin NINGUNA versión cacheada (mirar logs `[RD]`). |
| **Pipeline `/t/` — CDN mismatch** | `mediaInfos` da un CDN pero los segmentos del MPD pueden venir de otro (el `<BaseURL>` del MPD manda). | El navegador usa el `<BaseURL>` del MPD automáticamente; `mediaInfos.cdn` es solo referencia. Si no matchea, igual funciona (el CDN real está en el MPD). |
| **Pipeline `/t/` — `mediaId` inválido/expirado** | El streaming page o el `mediaId` pueden expirar o ser inválidos → 403 en segmentos. | Boot stage detecta error; fallback a transcode DASH legacy. |
| **Anime no reproduce en móvil (iOS Safari/WebKit)** | Error JW Player 102630 "This video file cannot be played". Solo falla en móvil (WebKit); desktop/TV (Blink) con el mismo Chrome sí reproducen. Hipótesis: servidores default usan HLS/MSE, restringido en iframes de terceros en iOS. | **NO RESUELTO.** Fix intentado 2026-06-30 (preferir MP4Upload/no-HLS por default en móvil, `pickDefaultAnimeServerIdx` en `PlayerView.vue`, commit `361a3e5`) **descartado por el usuario** — no solucionó el problema. Selector de servidores ya es visible en móvil (antes lo tapaba el CSS de fullscreen, fix `ab0e921`) para poder probar manualmente. Ver `sessions/2026-06-30.md §4`. |

---

## 4. Próximos pasos

1. ~~Confirmar reproducción RD con el token nuevo~~ ✅ HECHO (2026-06-09).
2. ~~Subtítulos/audio multi-audio (caso Weapons)~~ ✅ RESUELTO (rescate de versión cacheada).
3. ~~Arquitectura RD server-side (no Torrentio, token oculto)~~ ✅ IMPLEMENTADO (ADR-004; falta prueba end-to-end).
4. ~~Subtítulos de RD vs OpenSubtitles~~ ✅ DECIDIDO (ADR-005: OpenSubtitles).
5. ~~Optimización memoria #2 (buffer Shaka)~~ ✅ HECHO. (#1 content-visibility revertido a pedido del usuario.)
6. ~~Recuperación de stall al pausar (DASH)~~ ✅ HECHO.
7. ~~ADR-006 — borrar torrents de RD al cerrar~~ ✅ HECHO (function `rd-cleanup`).
8. ~~Deploy en vivo + git/GitHub/Netlify + SPA redirects (404)~~ ✅ HECHO.
9. ~~Fix TV: player pantalla completa, nav, buscador, tamaños, Magic remote, navegación D-pad~~ ✅ HECHO.
10. ~~Guard anti-archivo-basura (caso Scary Movie)~~ ✅ HECHO.
11. ~~Botón Reproducir (1 ícono, sentence case) + mensaje "no disponible" corto~~ ✅ HECHO.
12. ~~Fase 5 (servidor ffmpeg) — CANCELLED: reemplazado por pipeline `/t/` de RD~~ ✅ 🗑️
13. ~~Pipeline `/t/` de RD — INVESTIGADO, IMPLEMENTADO, DEPLOYADO~~ ✅ HECHO.
14. **Afinar pipeline `/t/`**: elegir CDN correcta, manejar 403, detectar `mediaId` inválido, auto-fallback.
15. Probar `rd-stream` end-to-end (título sin versión cacheada).
16. **Probar pipeline `/t/` en TV** (LG webOS) — puede tener limitaciones de DASH nativo.
17. Menú de filtros deslizante estilo Netflix (TV) — feature nueva propuesta.
18. Navegación D-pad (flechas) en tabs y píldoras de género (hoy solo puntero). Menor.
19. Optimización memoria #3 (capar grilla infinita en TV/móvil). Menor.
20. Empaquetado: PWA primero; luego webOS `.ipk` / Tizen `.wgt` / Capacitor `.apk`.
21. Portar Anime y Deportes al Inicio.
22. Continuar la migración hasta completar los 20 requisitos originales.
23. ~~Rediseño móvil estilo Netflix (hero/nav/detalle/fullscreen iOS)~~ ✅ HECHO (2026-06-29/30).
24. **Anime no reproduce en móvil (iOS)** — bug ABIERTO, ver Riesgos y `sessions/2026-06-30.md §4`.

---

## 5. Mapa rápido de "dónde tocar"

| Tarea | Archivo(s) clave |
|---|---|
| Lógica de reproducción RD (pipeline `/t/` + legacy DASH + Direct Play) | `src/composables/usePlayer.ts`, `useTpipeline.ts` |
| Pipeline `/t/` de RD (CDN, URLs, boot) | `src/services/rd-t-pipeline.ts`, `rd-media-infos.ts`, `rd-bootstrap.ts` |
| Player pipeline `/t/` | `src/components/player/TpipelinePlayer.vue` |
| Selección/scoring de stream RD | `src/services/streamSelector.ts`, `src/services/rdStream.ts` |
| Subtítulos (OpenSubtitles) | `src/composables/useSubtitles.ts`, `src/services/subtitles.ts`, `netlify/functions/subhash.cjs` |
| Huella del archivo (buscar el subtítulo de ESA copia) | `netlify/functions/moviehash.js`, `netlify/lib/rd.js` |
| Ajuste automático del subtítulo contra el audio | `netlify/lib/alinear.js`, `netlify/functions/alinear-background.js`, `alinear-estado.js` |
| Ajuste de sincronía compartido entre reproducciones | `netlify/functions/offset.js` |
| Panel de Sincronización (botones + y −, "ajustar al audio") | `src/components/player/PlayerSettingsPanel.vue` |
| Token RD | `src/services/bootstrap.ts`, `netlify/functions/rd-token.js`, `.env` |
| Inicio | `src/views/HomeView.vue` |
| Películas / Series | `src/views/MoviesView.vue`, `src/views/SeriesView.vue` |
| Grilla "Ver todo" | `src/views/AllResultsView.vue` |
| Carrusel / cards | `src/components/catalog/Carousel.vue`, `MovieCard.vue` |
| Nav superior | `src/components/layout/NavDesktop.vue` |
