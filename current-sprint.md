# Current Sprint — MaldoxFilm

> Tablero del sprint vigente. Actualizar al finalizar cada tarea.
> Última actualización: 2026-10-12

**Objetivo del sprint:** reproducción sin cortes. Ver `adr/ADR-010.md`.

---

## 🟢 ÚLTIMO PUNTO ESTABLE CONOCIDO

> Si el usuario pregunta "¿cuál era la versión que funcionaba?", la respuesta es esta.
> No es una rama: es una ETIQUETA sobre `main`. La rama siempre es `main`.

**`estable-2026-10-18-noche`** → commit **`a8ce410`** · domingo 18 de octubre de 2026, de noche

Verificado en vivo por el usuario en la TV, punto por punto:

- Pausa de **24 minutos** y la película siguió directo, sin un solo aviso
- Corte real del enlace en *Tenzing*: reconectó y retomó en 6 segundos
- Botón "ajustar al audio": midió -14,2 s en *The Dark and the Wicked*
- La cadena de reproductores va una vez por fuente y termina en el cartel
- Buscador de TV con botón de limpiar

418/418 tests, `vue-tsc -b` limpio, `vite build` OK, desplegado en producción.

Los anteriores siguen disponibles: **`estable-2026-10-17-noche`** → `1970512`
(antes de todo lo de subtítulos y pausas) y **`estable-2026-09-04`** → `2a6d7d0`.

Volver al punto estable:

```bash
cd /Users/angelomaldonado/Desktop/maldoxfilm-vue && git reset --hard estable-2026-10-18-noche && git push origin main --force
```

Y redesplegar:

```bash
cd /Users/angelomaldonado/Desktop/maldoxfilm-vue && npx vite build && netlify deploy --prod --message "rollback a estable-2026-10-18-noche"
```

---

## ▶️ RETOMAR AQUÍ (próxima sesión)

**Estado:** app en vivo y deployada. HEAD `3c5e73c`. **408 tests verdes**, `vue-tsc -b`
OK, build OK. Código en `/Users/angelomaldonado/Desktop/maldoxfilm-vue`.

### Hecho el 2026-11-05 (ver `sessions/2026-11-05.md`)

- `632585a` — **Quinta forma de reconocer el archivo en la cuenta de RD**, por
  nombre normalizado. En *Kids* (1995), 9 de 34 copias venían con
  `filename: null` y eran irreconocibles: la película no arrancaba estando el
  archivo en la cuenta.
- `5431a34` — **vidlink y vidsrc no se ofrecen en la TV**: ahí dejan la pantalla
  en blanco y congelan la página entera. En escritorio siguen.
- Documento nuevo con el flujo completo:
  [`architecture/flujo-reproduccion.md`](architecture/flujo-reproduccion.md).

### Hecho el 2026-10-20 (ver `sessions/2026-10-20.md`)

- `ce31692` — **La medición del subtítulo ya no se rinde por poco.** *Hope*
  (coreana) daba margen 1,245 contra un corte de 1,25 y no aplicaba nada;
  escuchando el doble de tramos da -15,3 s con margen 1,30. Ahora, cuando la
  primera pasada queda al borde, hace una segunda sola. Verificado en la TV:
  aplicó -15,6 s en dos minutos. **No es cuestión de idioma** (la francesa dio el
  margen más alto de todas), sino de cuánta voz tiene la película.

### Hecho el 2026-10-17 (ver `sessions/2026-10-17.md`)

- `6816194` + `29368ad` — Un códec que el navegador no muestra (mpeg4/DivX/Xvid y
  compañía) ya no se intenta en directa: se cambia de reproductor. Era lo que
  distorsionaba la imagen con *Scrapbook*.
- `1728d28` — **La TV ya no se detecta como celular.** El navegador de los LG dice
  `Web0S` con un cero y esa variante faltaba, así que todo dependía de la
  heurística de tamaño/táctil, que en webOS no da siempre lo mismo. Cuando fallaba
  salían los iconos de móvil abajo y el layout se descuadraba. El modo `tv` queda
  anotado en `localStorage` y no se degrada en cargas posteriores.
- `3c5e73c` — **Se acabó el carrusel RD → UnlimPlay → RD.** Cada fuente se intenta
  una vez por título, la cadena avanza hacia adelante y, agotadas las tres, sale
  "Película no encontrada" con botón Aceptar. Además: el paso de UnlimPlay a
  vidlink no ocurría nunca (el `about:blank` del iframe disparaba `onload` y se
  tomaba por carga buena) y la X parecía pegada por las entradas de historial que
  dejaba cada carga de iframe.

- `6efb84a` — **Encontrada la causa real del bucle, medida en la TV.** La página
  embebida navegaba la ventana de arriba y recargaba la app entera cada seis
  segundos (`carga:reload` sobre `/ver/movie/59317` en el log), y cada arranque
  reiniciaba el título desde RD. El iframe lleva ahora `sandbox` sin
  `allow-top-navigation`, y las fuentes descartadas se anotan tres minutos en
  `sessionStorage` para que ninguna recarga reinicie el recorrido.

- `f57b8c4` — **Botón "ajustar al audio"**: mide el desfase del subtítulo contra
  el audio y lo corrige solo, en alrededor de un minuto. Validado en tres
  películas (dos corridas, una sana) y verificado en producción. Detalle y lista
  de lo que ya se probó y no sirve en `sessions/2026-10-17.md`.

- `ceeb310` · `a8ce410` · `5590c58` — **Tolerancia a las pausas largas.** El
  enlace de Real-Debrid se mantiene vivo mientras la película está pausada
  (verificado: 24 minutos y siguió directo); si igual se corta, reconecta sola y
  retoma donde estaba; y al terminar la película los controles quedan a la vista
  para poder volver.

**Ojo con la TV:** su identificador de navegador CAMBIA entre cargas, `NetCast /
Colt` en el primer arranque y `Windows / Whale` después de una recarga. Es el mismo
televisor. No confiar solo en la detección por marca.

- `38b7489` — **Los subtítulos se buscan por la huella del archivo**, no solo por
  su nombre. Nueva function `moviehash`. Un subtítulo que calza con la huella fue
  subido para esa copia exacta, así que viene sincronizado; además la huella
  revela el nombre real del release, que se usa para la búsqueda en español y
  para las pistas de calidad. Verificado en producción.

**Por verificar:** que `sandbox` no le moleste a UnlimPlay ni a vidlink. Si alguna
dejara de reproducir, el token que falta sería `allow-popups`, nunca
`allow-top-navigation`, que es justo el que producía el bucle.

### Hecho el 2026-10-12 (ver `sessions/2026-10-12.md`)

⚠️ **Lo primero, porque es lo que más tiempo costó:** el repo estaba enlazado al sitio
de Netlify EQUIVOCADO (`funny-jelly-2afcd4`, ya eliminado) y el build automático de
`maldoxfilm` llevaba roto desde el 11-sep. Ningún arreglo llegaba al sitio del usuario
mientras la verificación daba todo por correcto, porque consultaba la URL del CLI y no
la que el usuario usa. **Verificar siempre contra `maldoxfilm.netlify.app`.**

- `c6bec0b` — **Build automático reparado.** `NAV_ITEMS_TV.length` dentro de la
  plantilla de `NavTV.vue` hace fallar a `vue-tsc` en Netlify ("Property 'value' does
  not exist on type 'NavItem[]'") aunque pase limpio en local. Los índices se calculan
  ahora en el script. Se fijó además `NODE_VERSION = "20"` en `netlify.toml` (no era la
  causa, pero depender de la versión por defecto es frágil).
- `9da3e64` — **En TV no se convierte audio nunca.** Si el latino sale directo, latino;
  si no, inglés con subtítulos. Lo que no se acepta es que se corte.
- `7423c27` — **El panel informa la pista que suena de verdad**, no lo que diga el
  nombre del archivo. Antes `...ENG.LATINO.HINDI...` hacía mostrar "Audio: Español" y
  apagaba los subtítulos mientras sonaba inglés.
- `ac4d1e2` — Entre copias latinas gana la que lista el español primero en el nombre
  (+60), porque en directa suena la primera pista. Solo desempata dentro de la misma
  clase de reproducción.
- `f6a73d3` — Mensaje propio cuando falta `RD_TOKEN`, en vez de "Sin resultados en RD".

**TorBox cerrado:** el plan Pro venció, la API ya no responde. Quedan dos torrents de
prueba por borrar a mano desde su panel.

### Hecho el 2026-09-11 (ver `sessions/2026-09-11.md`)

- `b1c9183` — **La TV decodifica AC3 directo: se acabaron los cortes en el
  televisor.** Comprobado en vivo: la misma TV dice NO por `MediaSource` y
  `"probably"` por `canPlayType`, y reproduce un h264 + AC3 real CON audio sin
  convertir. La regla excluía AC3 para todos porque se escribió desde el fallo en
  escritorio, así que en la TV se mandaban a convertir archivos que ahí se
  reproducen directos. Ahora se decide por capacidad (`detectAc3Support`, exige
  `"probably"`), no por dispositivo. Escritorio sin cambios.
- **TorBox evaluado y DESCARTADO** con seis mediciones (0,51x a 0,84x, mismo
  rango que RD 0,70x) y confirmado por el propio reproductor web de TorBox, que
  se corta cada 5 segundos con su propio transcode. **No migrar.**
- **`luna://com.webos.media`** (el motor nativo de la TV, lo que usa Stremio en
  LG) decodifica todo sin convertir, pero `window.webOS` NO existe en el
  navegador de la TV: exigiría empaquetar la app como aplicación webOS. Opción
  futura, no descartada.

### Hecho el 2026-09-04 (ver `sessions/2026-09-04.md`)

Cuatro correcciones, todas verificadas contra el build real y confirmadas en vivo:

- `c6aacf1` — **TV: el layout ya no colapsa.** En fullscreen webOS deja el viewport de
  layout en un ancho degenerado y todo se partía una palabra por línea (se veía entero
  en "Mi Lista" vacía). Piso de ancho `html.tv-mode #app-shell` alimentado por
  `--tv-screen-w` (`screen.width`, que no se degrada). La mitigación que había en
  `App.vue` (dispatch de `resize`) **no hacía nada**: no hay un solo listener de
  `resize` en la app. Se dejó por inocua, con el comentario corregido.
- `3372f5c` — **Buscar responde al primer intento en TV.** Se precarga el chunk de
  `/buscar` (import dinámico) al quedar el navegador ocioso, y se dejó de pedir
  `/trending/all/week` fuera de móvil, que era el único lugar donde se renderiza.
- `871a439` — **El buscador ya no parpadea al escribir.** El spinner solo aparece si
  todavía no hay resultados; antes borraba los visibles en cada tecla.
- `2a6d7d0` — **"Ver todo" devuelve a la posición al volver.** Esa vista no scrollea la
  ventana (`position: fixed` + `overflow-y: auto` propio), así que el `scrollBehavior`
  del router nunca pudo restaurarla. No era regresión: nunca estuvo cubierta por
  `f8edf81` (09-ago).

⚠️ **Para quien toque `AllResultsView` o cualquier restauración de scroll:** este router
NO escribe `state.key` (solo `{back, current, forward, replaced, position, scroll}`), y
`position` se REUTILIZA entre entradas distintas. La dirección de la navegación se
distingue con `forward` (null en entrada nueva, con valor al volver). Todo comprobado
en el navegador; resolverlo de memoria daba código que no guardaba nada.

### Lo que se resolvió el 2026-08-02 (ver `adr/ADR-010.md`)

La causa real de los cortes NO era el pipeline `/t/`: era que se decidía usarlo
leyendo el NOMBRE del archivo. Copias que ya estaban en h264+aac se mandaban a
convertir a RD, que las generaba a 0.70x. Ahora se consulta `mediaInfos` y el códec
REAL decide. Verificado en vivo: latino, 1080p, Direct Play, far-seek instantáneo.

También se ELIMINÓ por completo el cambio automático de copia (Plan B), a pedido
explícito del usuario. **No reintroducirlo.**

### Dos sitios en Netlify — OJO con esto (verificado 2026-08-03)

- **`maldoxfilm.netlify.app`** → conectado a GitHub. **Es la producción.** Deploya SOLO
  con `git push origin main`, automáticamente. Confirmado funcionando.
- **`funny-jelly-2afcd4.netlify.app`** → secundario, se actualiza solo por CLI.

**El repo local está enlazado al SEGUNDO.** Por eso `netlify dev`, `netlify deploy` y
`netlify status` desde esa carpeta apuntan al sitio equivocado. Ya causó un diagnóstico
errado ("el auto-deploy no funciona" — sí funciona, se estaba mirando el otro sitio).
Para arreglarlo: `netlify link --name maldoxfilm`.

Para verificar que un deploy llegó, NO sirve pedir un asset y mirar el código HTTP: el
fallback de la SPA devuelve 200 para cualquier ruta, incluso inventada. Hay que bajar el
bundle y buscar un marcador del código nuevo.

### Hecho y DEPLOYADO (2026-08-02 → 03)

- `e7bee2c` — el códec REAL (`mediaInfos`) decide sobre el nombre. Causa raíz de los
  cortes. Plan B eliminado por completo (463 líneas, NO reintroducir).
- `e7718ca` — `CAMBIOS_2026-08-02.md` en la raíz del repo, resumen para el usuario.
- `d187148` — clases de reproducción en la selección (`playbackClass` + `rankStreams`
  ordenando por clase y después por puntaje). Sin tocar pesos del scoring.
- `e71038f` — `getGb` no sabía leer MB (todo archivo de menos de 1 GB caía al default de
  5 GB, justo el tramo premiado con +40) + tope de 20 GB para la clase A.
- `3af7912` — adenda en `CAMBIOS_2026-08-02.md`.

**TAG: `hito-reproduccion-fluida-2026-08-03`** — punto de referencia del trabajo. Ahí
quedó resuelto el problema de fondo y verificado en vivo con 5 películas.
Para volver a este estado: `git checkout hito-reproduccion-fluida-2026-08-03`.

Verificado en producción bajando el bundle y buscando marcadores del código nuevo (NO
sirve pedir un asset y mirar el código HTTP: el fallback de la SPA devuelve 200 siempre).

Verificado en vivo con dos películas: Sueños de Fuga (estaba rota → latino 1080p Direct
Play, far-seek instantáneo) y Parásitos (andaba bien → sigue igual). 354 tests.

### Pendiente inmediato

-1. **La TV expone `audioTracks`** (informó las 3 pistas del archivo de prueba).
   Hoy el cambio de idioma solo funciona por el camino de conversión; con eso
   quizás se pueda ofrecer también en reproducción directa.
0. **"No se reproduce" desde Ver todo → Suspenso** (reportado 2026-09-04, SIN
   resolver). Se verificó en vivo que el click SÍ abre la ficha, y ningún cambio de esa
   sesión toca el click ni la reproducción. La sospecha es que esa fila está llena de
   estrenos 2026 sin copias en RD, que se ve igual que un bug sin serlo. **Falta el
   nombre de un título concreto** para rastrearlo. No se probó la reproducción desde la
   sesión a propósito: hacerlo agrega torrents a la cuenta RD del usuario.
1. **Probar más títulos**, sobre todo alguno cuya copia SÍ necesite conversión de verdad
   (AC3/DTS/HEVC reales), que es donde las clases recién ahora hacen diferencia. Y probar
   en TV, que no se tocó en toda la jornada.
2. **`findCachedByTitleYear`** recorre el historial de la cuenta ignorando el scoring y
   puede adoptar una copia de clase B habiendo una de clase A. Ya rompió un fix el 27-jul
   por lo mismo. Debería correr solo si no hay clase A.
3. **El guard de idioma del "upgrade a fluido"** (`resolveActiveStream`) usa
   `isDirectPlayEligible`, el criterio estricto que ADR-010 desmiente. Hoy es inocuo
   porque la clase decide antes, pero deja dos políticas opuestas en el mismo archivo.
4. **Aviso al usuario** cuando la copia elegida no sea latina existiendo una latina en el
   catálogo (reusar el mecanismo de `foreignAudioLanguage`).

### Trabajo descartado que quedó guardado

`git stash@{0}` — tres fases sobre `/t/` (re-siembra de base de tiempo, colchón
post-salto, gate de arranque por ritmo, caída a Direct Play). Verificado con tests
pero REVERTIDO tras fallar en vivo. Si alguna vez se retoma, revisar primero por qué
falló la caída a Direct Play (había una carrera con el teardown de Shaka, ya
corregida por separado en `_shakaDetachAndWait`).

---

## Historial del sprint (entradas anteriores)

**Estado:** app EN VIVO (`maldoxfilm.netlify.app`), git→GitHub→Netlify (push a `main` = deploy).
**HEAD `fe7c6f4`**. **312 tests verdes**, `vue-tsc -b` OK, build OK. Working tree limpio.
Código en `/Users/angelomaldonado/Desktop/maldoxfilm-vue`.

⚠️ **Dos sesiones de IA trabajaron EN PARALELO la noche 13→14-jul** sobre este
repo (sin colisión, por suerte + verificación manual del HEAD). Si hay más de
una sesión activa: `git log -1` antes de editar, y verificar el remoto antes de
cada push. Los 14 commits de la "otra sesión" (rd-stream por fases, /t/ sin
rdId, rescate título+año, subtítulos) + el re-fetch del pool están todos en
`fe7c6f4`. Ver `sessions/2026-07-14.md`.

### ✅ Sesión 2026-07-14 — Anti-pausas en dos frentes [COMPLETADO, deployado] (ver sessions/2026-07-14.md)
- **Re-fetch del pool (`fe7c6f4`):** caso real "Ghost Rider" — pool pobre (19
  streams) sin versión fluida → /t/ sobre archivo que RD generaba a 0.42x
  (prueba matemática en los latidos del log) → pausas inevitables. Fix: 2da
  consulta a Torrentio cuando la elegida no es Direct Play + re-selección.
- **Plan B en /t/ (`2a99728`):** a la 2da recuperación de stall sobre la misma
  copia (lentitud confirmada), cambiar automáticamente a la siguiente copia
  cacheada desde la misma posición (`altCachedCandidates`, guard de idioma) en
  vez de recargar la lenta hasta caer al iframe. HEAD final: `2a99728`, 314 tests.
- **Subtítulos quemados (`aff3612`):** caso real "La muerte de Robin Hood" — dos
  subtítulos superpuestos (lituano quemado en la imagen + el nuestro en español).
  El tag `HC` (hardcoded) no se miraba en el scoring. Fix: -150 puntos (P8),
  calibrado con los puntajes reales del log; no rompe "cacheado gana" ni descarta
  si HC es la única opción. HEAD final: `aff3612`, 318 tests.
- **Subtítulos quemados — 3 iteraciones hasta cerrarlo (`d206dc9`, `3216298`):**
  penalizar `HC` no bastó (otra etiqueta `PLSUBBED` ganó); generalizar la detección
  tampoco (el RESCATE ignora el scoring y busca directo en el historial de
  descargas de la cuenta). Fix final: ambos rescates saltan copias quemadas
  cuando hay alternativa limpia en el catálogo general, dejando `rdId` null a
  propósito para que `usePlayer` derive el id del link crudo. HEAD: `3216298`,
  324 tests. Lección: un scoring corregido no sirve si hay rutas que lo bypassean.
- **Salto a otro reproductor si nada es confiable (`04e7669`, 4ta vuelta):** 3ª
  copia distinta confirmada con el mismo subtítulo quemado (sin declarar nada) →
  se agotó el whack-a-mole por nombre. Ahora `noTrustworthyCachedVersion` salta
  directo a UnlimPlay/vidlink cuando NINGÚN candidato cacheado declara estar
  libre de riesgo (ni HC/región ni origen cine). HEAD: `04e7669`, 328 tests.
  **Límite reconocido:** no detecta copias sin ninguna etiqueta que igual estén
  contaminadas (necesitaría OCR). Próximo paso si se repite: reporte manual del
  usuario para descartar un rdId específico, no más regex de nombre.
- **Botón "reportar copia" — implementado y luego REVERTIDO a pedido explícito
  del usuario** (`c44b61b` → revertido en `1f486be`): quería solo automatización,
  sin controles manuales. HEAD final: `1f486be`, 328 tests, sin botón.
  **Estado final de detección de subs quemados: 3 capas automáticas** (penalizar
  tag declarado, rescate no adopta contaminadas si hay alternativa, salto a otro
  reproductor si NADA es confiable). Techo real aceptado: un archivo que MIENTE
  en su nombre (dice "WEBRip" sin serlo) no es detectable sin OCR — descartado
  por riesgo de falsos positivos. **No intentar más regex ni controles manuales
  para este defecto en sesiones futuras — ya se agotaron 4 rondas + 1 rechazo
  explícito del usuario.**
- **Cirílico no detectado (`9eb405e`, 5ta vuelta) — bug REAL, no límite filosófico:**
  la misma copia rusa (id `UT3KNP27TQQ6Y`) volvía siempre porque `hasBadLang` buscaba
  "rus"/"russian" en LATÍN y el título venía en cirílico. Fix: `hasNonLatinTitle`
  (cirílico/griego/CJK/hangul/árabe/hebreo) sumado a `hasBadLang` → -10000 automático.
  HEAD: `9eb405e`, 333 tests. 100% automático, sin controles manuales.
- **Gate de idioma original vía TMDB (`297050e`):** caso real "Kraken" — película
  NORUEGA (confirmado TMDB/Wikipedia), copia elegida sin ninguna etiqueta de
  idioma. Nuevo `TmdbClient.getOriginalLanguage` + gate en `rdStream.ts`: si el
  idioma original no es en/es, exige audio confirmado (ENG/SPA/Latino) o salta a
  otro reproductor (`noConfirmedLanguageMatch`). NO afecta Hollywood (idioma
  original ya es "en", gate nunca se activa). HEAD: `297050e`, 338 tests. 100%
  automático.
- **Verificación de duración en subtítulos (`9266dce`):** el "piso de confianza"
  (ya en prod, otra sesión) protege contra subs de OTRA película, no contra la
  MISMA película en CORTE distinto (caso Robin Hood: WEB-DL srt en video rip de
  cine). Nuevo: si la duración implícita del `.srt` no calza con `video.duration`
  (margen 600s), prueba el siguiente candidato; si ninguno calza, usa el mejor
  igual (con log). Límite reconocido: no resuelve un offset constante por intros
  distintas de igual duración total — para eso sigue el ajuste manual +/-5s.
  HEAD: `9266dce`, 344 tests.
- **Audio extranjero: aviso, no bloqueo (`73c5e0d`):** el usuario reconsideró el
  gate de idioma — ahora Kraken (y similares) reproducen en su audio original con
  toast `🌐 Audio en {idioma}` + subtítulo en español normal (busca por IMDB, no
  depende del audio). `noConfirmedLanguageMatch` → `foreignAudioLanguage`.
  HEAD: `73c5e0d`, 344 tests.
- **Plan B no resincronizaba subtítulos (`1f19623`):** usuario reportó desfase
  sin dirección fija (a veces atrasado, a veces adelantado) — causa real: el
  Plan B cambia de copia (video distinto) pero nunca re-avisaba al sistema de
  subtítulos, que seguía con el `.srt`/offset de la copia ORIGINAL. Fix:
  re-dispara `onStreamReady` con el filename de la copia nueva tras el swap →
  nueva búsqueda + verificación de duración contra el archivo real. NO se quitó
  el Plan B (resuelve el problema real de copias lentas). HEAD: `1f19623`, 344
  tests.
- **Bug propio del fix anterior corregido (`02b1181`):** tras `1f19623`, el
  usuario reportó "el subtítulo sale antes de hablar" — el re-disparo de
  `onStreamReady` para el Plan B llamaba `fetchAndInject` sin pasar por
  `clear()`, así que si no había offset guardado para el archivo nuevo, se
  arrastraba el ajuste MANUAL de la copia vieja. Fix de una línea: `offsetMs`
  siempre se resuelve explícito (guardado o 0). HEAD: `02b1181`, 344 tests.
  Lección: un fix que reusa código en un flujo nuevo puede exponer supuestos
  implícitos — revisar el flujo completo, no solo el síntoma original.
### ⭐ CAUSA RAÍZ de las pausas encontrada (`cb0fc72`) — descarga paralela de segmentos

Shaka descargaba segmentos **secuencialmente** (`segmentPrefetchLimit` default = 0,
confirmado en doc oficial). Contra un CDN da igual, pero `/t/` de RD **genera bajo
demanda** → RD quedaba OCIOSO entre pedidos. Evidencia medida (mismo archivo, mismo
servidor): pre-buffer **1.75x** vs reproducción **0.42x**. Fix: `segmentPrefetchLimit: 4`.
Explica por qué los experimentos de buffer del 13-jul fallaron (atacaban el síntoma).

**Métrica nueva para verificar:** el latido ahora loguea `RD genera: N.NNx` (+ `⚠️ DÉFICIT`
bajo 1.0x). **Nunca volver a tocar buffer/prefetch sin mirar ese número.**
Tag de rollback: `pre-segment-prefetch`. HEAD: `cb0fc72`, 344 tests.

**Siguientes pasos si no basta** (ver detalle en sessions/2026-07-14.md):
1. Disparador temprano por pre-buffer (el dato YA se mide y hoy se ignora — Robin Hood
   0.25x se pegó, Kraken 1.25x no se pegó: es un predictor real y gratis).
2. Sumar TorBox como segundo pool de caché (Torrentio soporta `torbox={TOKEN}`).
3. Subir prefetch 4→6, pero SOLO midiendo con la métrica nueva.

### ⚠️ Prefetch paralelo (`cb0fc72`) DESCARTADO como causa de fondo — medido, no supuesto

Con la métrica nueva se probó Ghost Rider: en PAUSA (sin límite de consumo del
cliente) RD generó constante ~0.83x. Si el prefetch hubiera sido la causa, en pausa
el ritmo debería subir — no subió. Es un LÍMITE DE GENERACIÓN REAL del servidor de
RD para ese archivo, no cuántos segmentos pide el cliente. El fix queda desplegado
(no hace daño) pero descartado como explicación. Lección: la métrica sirvió
exactamente para esto — desmentir una hipótesis razonable con datos.

### ⭐⭐ Plan B blindado + PREVENTIVO (`001e140` + `c85d8b9`) — "solución definitiva" pedida

Usuario pidió no esperar a que se trabe. Con el ritmo real medido (0.94x sostenido
→ igual se pegó pese a 40s de buffer), se agregó un disparador PREVENTIVO: si el
promedio de 5 latidos (~30s) reproduciendo da bajo 0.95x, cambia de copia ANTES de
que se note (función `tryPlanBSwap` extraída y reusada por ambos disparadores,
preventivo + reactivo). Blindado en 2 capas contra saltar de corte/edición
(Extended↔Theatrical): `cutMarker()` por nombre + verificación de duración real
(margen 180s) cuando RD resuelve el archivo. HEAD: `c85d8b9`, 348 tests.

**Límite honesto que sigue en pie:** si TODAS las copias cacheadas generan lento
ese día, ningún cambio de copia lo arregla (le pasa hasta al player oficial de RD).
Se redujo drásticamente cuántas veces se llega a ese techo, no se eliminó el techo.

Tags de rollback disponibles: `pre-segment-prefetch`, `pre-plan-b-preventivo`.

### Caso real "Doctor Sleep" — confirma el límite (no un bug) + fix de detección (`8cf4ac2`)

Se pegó pese a los fixes de hoy. Log analizado: pre-buffer YA salía en 0.25x desde
el arranque, latidos con varios `0.00x` reales. **El Plan B nunca tuvo con qué
actuar** (`alts.length === 0` — nunca apareció el log de alternativas): solo puede
cambiar a copias YA en el historial de descargas de la cuenta, y no había ninguna
otra edición de este título descargada antes. No es un bug de los fixes de hoy —
es el límite estructural del Plan B (depende de qué ya está en la cuenta).
Bug real corregido de paso (no la causa): `cutMarker` no reconocía "DC" como
abreviación de Director's Cut. HEAD: `8cf4ac2`, 350 tests.

### ⭐⭐⭐ CAUSA ESTRUCTURAL RESUELTA (`d3263ae`) — el Plan B exigía historial previo

El Plan B solo consideraba copias con `matchInDownloads` (ya reproducidas antes por
esa cuenta) → Doctor Sleep tenía **11 candidatos `[RD+]` y CERO alternativas**.
Restricción ARTIFICIAL: `[RD+]` = cacheado en el pool GLOBAL de RD, y resolverlo lo
agrega a la cuenta en segundos (mismo camino que usa la copia principal). La función
`resolveRawToRdId` YA existía en el repo sin conectarse a este flujo.

Ahora `altCachedCandidates` acepta `{rdId}` (instantáneo) o `{url}` (se resuelve
ON DEMAND solo si el Plan B lo necesita → cero costo/cuota si va bien). Guards de
idioma/corte/duración aplican igual. `[RD download]` sigue excluido (RD tendría que
descargarlo). **Esto vuelve innecesaria la "precarga en segundo plano" propuesta
antes** — se resolvió mejor: bajo demanda. HEAD: `d3263ae`, 351 tests.

**Lección:** cuando el usuario diga "es un parche, busca la causa real" → releer el
flujo buscando RESTRICCIONES DE DISEÑO sin justificación técnica, no ajustar
umbrales. Acá la infraestructura correcta ya existía, solo faltaba conectarla.

### Primera activación real del Plan B preventivo + fix del resolve (`e1d3e3b`)

Log del usuario (Doctor Sleep) confirmó: alternativa encontrada donde antes había
CERO ✅, disparo preventivo funcionó ✅ (`promedio 0.79x en 5 latidos`), cutMarker
"DC"↔"DIRECTORS.CUT" emparejó bien ✅ — **pero falló al resolver la alternativa**.
Causa: `handleResolveRaw` consultaba `/downloads` una sola vez inmediatamente, y la
entrada recién agregada por Torrentio puede tardar en aparecer. Fix: 3 reintentos
espaciados 1.2s + diagnóstico real en el log (antes fallaba en silencio).

### ⭐ RESUELTO — el criterio de disparo mide MARGEN REAL (`05ea71c`)

Log de 5 min continuos (Doctor Sleep): **CERO `[PLAYER] waiting`**, buffer 23s→70s→56s.
La reproducción ya no se corta. Pero el log mostraba `⚠️ DÉFICIT` constante porque RD
genera POR RÁFAGAS (0.00x→2.51x→0.00x→3.34x) y se marcaba cada lectura suelta.

Corregido: el ritmo aislado NO predice cortes. Ahora se usa
`margen = buffer / (1 - ritmoPromedio)`:
  56s @ 0.95x → 1120s de margen → no tocar nada (antes habría bajado a 720p al pedo)
  10s @ 0.50x → 20s → cambiar YA
`PLAN_B_TIME_TO_EMPTY_SEC = 90` reemplaza a `PLAN_B_RATE_THRESHOLD`. El log ahora
muestra promedio + margen real y solo alerta cuando hay riesgo verdadero.
**Esto cierra la disyuntiva 30s vs 60s** — el criterio pasó de temporal a predictivo.
HEAD: `05ea71c`, 351 tests.

- Pendiente de validación del usuario: **reproducir un título que antes se pegaba y
  confirmar si el Plan B preventivo dispara** (buscar en el log `Plan B PREVENTIVO
  — déficit sostenido` y `✅ Plan B activo`) — esa es la prueba real de si esto
  redujo las pausas; decidir si se implementa la precarga en segundo plano;
  confirmar que el subtítulo no sale adelantado tras un swap; probar "Kraken"
  (toast + subtítulo español con audio noruego); y hacer un SEEK real (verificado
  por código, no en vivo).

⚠️ **Nota de contexto crítica:** hoy hubo un rollback de emergencia que llegó
demasiado lejos (`04f1d73`, 27-jun) y borró ~2 semanas de trabajo válido — se
recuperó moviendo `main` a `de90173` + los 2 fixes nuevos. Ver detalle completo en
`sessions/2026-07-13.md` y `project-memory.md §0.0` antes de asumir que el HEAD
actual es continuación lineal de `a664f9a`/`4eaca8e` (esos fueron puntos
intermedios de rollback, ya superados).

### 🔲 Pendiente — análisis de arquitectura (prompt ya armado para Fable 5)

- Rediseño de `streamSelector.ts`: detección de audio real (no solo nombre de
  archivo) + balance idioma-latino (+200) vs. confiabilidad real de reproducción.
- Rediseño del flujo sin `rdId` (`rd-stream.js`): evaluar Netlify Background
  Function (sin el techo de ~10s síncrono) + fallback intermedio más seguro que
  el link crudo de Torrentio.
- Revisar lógica de "seek freeze/bounce-back" — hoy genérica para todo camino
  no-`/t/`, incluyendo el fallback crudo.
- Confirmar/investigar bug de subtítulo mal emparejado (`score: 19`, título
  ajeno tipo Maze Runner) visto en un log de esta sesión.

### ✅ Sesión 2026-07-13 — Heartbeat validado, crisis de buffer, rollback recuperado, 2 fixes reales [COMPLETADO] (ver sessions/2026-07-13.md)

### ✅ Sesión 2026-07-06 (continuación) — TV: fullscreen + 3 bugs de pulido [COMPLETADO] (ver sessions/2026-07-06.md, sección "Continuación")
- Fullscreen ya no se pierde al entrar a reproducir (se reafirma en `init()`); el
  auto-activado en el PRIMER click de la app **no funcionó** y queda pendiente (ver
  abajo) — el usuario acepta apretar el botón del NavTV una vez por sesión.
- Flash negro al presionar "Volver" durante la reproducción — 2 causas reales
  encontradas y arregladas (vaciado prematuro del `<video>`, y luego `pause()`
  pintando negro en el decoder de hardware de la TV).
- Barra de reproducción (`.nf-controls`) que a veces quedaba pegada — tenía su propio
  auto-hide que nunca se armaba solo, arreglado.
- "La pantalla se agranda un poco" al dar Reproducir/Continuar — efecto colateral del
  fix de fullscreen (pedía `requestFullscreen()` sin chequear si ya estaba activo).
- HEAD `a664f9a`, 3 commits, 3 tags de rollback, deploy verificado con la API de
  Netlify en cada uno.

### ✅ Sesión 2026-07-06 — Maratón TV: 6 bugs reales resueltos [COMPLETADO] (ver sessions/2026-07-06.md)
Los congelamientos reportados ("se pausaba a cada instante", "queda cargando en TV") NO
eran (solo) el límite de RD generando `/t/` en vivo — eran bugs reales del código:
- ⭐ **Doble-resume**: al retomar "continuar viendo", un seek redundante tiraba el buffer
  ya acumulado y colgaba RD regenerando. **Causa principal de los congelamientos** —
  confirmado por el usuario tras el fix: "ahora no se ha pausado".
- ⭐ **`video.play()` colgado para siempre en TV** (webOS no resuelve esa promesa) —
  encontrado con un badge de diagnóstico en pantalla fotografiado en la TV.
- Timeout/onerror en carga de Shaka/HLS.js desde CDN, barra al 100% (duración mal
  reportada por RD), ícono de volumen desincronizado, SW fantasma, barra con % viejo
  al cambiar de título.
- **Enfoques descartados** (documentados para no repetir): subir buffer, `full.mp4`,
  freeze-frame en cortes, pre-roll, pre-cacheo inteligente (RD rechaza por DMCA),
  `/t/`-solo-para-H264+AAC (RD también se corta en su propio player con ese archivo).
- **Límite aceptado**: el tironeo intermitente de `/t/` en archivos sin H264+AAC
  (ej. "From") sigue existiendo — confirmado con el reproductor OFICIAL de RD, mismo
  archivo, mismo problema. No es del cliente, no se puede eliminar.

### 🗺️ ROADMAP ACORDADO (prioridad real)
1. **🔴 #1 — Validar UH8500** — capa de compatibilidad en TV vieja (Chromium 38). Pendiente.

2. **🔴 #2 — Anime no reproduce en móvil (iOS)** — bug crítico abierto desde 2026-06-30.
   Error JW Player 102630 en Safari/WebKit. Último fix (preferir MP4Upload) no funcionó.
   Requiere investigación externa o repensar embed vs. URL directo.

3. **🔴 #3 — Seguridad → Auth gate (~0.5 día)** — protege tu RD personal.
   App pública + RD personal = riesgo baneo/DMCA. Solución: password/PIN + token JWT.

4. **PWA fase 2** (después de #3): sincronizar progreso al servidor.

5. webOS `.ipk` / Capacitor (cuando anteriores estén ✅).

### ⏳ Pendiente menor
- ~~Sacar el badge de diagnóstico temporal (`onDiag`, `.tdiag-badge`)~~ — **ya se sacó**
  (commit `106da5a`, confirmado el bug de `video.play()` colgado). No queda rastro en
  `usePlayer.ts`/`VideoPlayer.vue`.
- **Auto-activar fullscreen en TV sin tocar el botón del NavTV** — el enganche al primer
  click/tecla de la app (`App.vue` `onMounted`) no funcionó (confirmado por el usuario).
  Sin acceso a consola en la TV no se pudo diagnosticar el motivo real del rechazo del
  Fullscreen API. Si se retoma: primero instrumentar el `.catch()` de `requestFullscreen()`
  con un toast visible para ver el error real antes de intentar otro fix a ciegas.

### ✅ Sesión 2026-07-01 — Capa de compatibilidad TVs viejas (Chromium 38) [COMPLETADO, deployado] (ver sessions/2026-07-01.md)
Para que la LG **UH8500** (webOS 3 = Chromium ~38) deje de distorsionar el CSS, SIN tocar
lo que anda en moderno. 3 pasos, cada uno verificado (type-check + 289 tests + build):
- **`d0d046e` (1/3)**: `@vitejs/plugin-legacy` + `terser` → 2º bundle ES5 + polyfills (JS).
- **`b255340` (2/3)**: `postcss.config.js` (`@csstools/postcss-global-data` + `postcss-custom
  -properties preserve`) → aplana los 291 `var(--)` a valor fijo (Ch38 no soporta variables CSS).
- **`4eaca8e` (3/3)**: `autoprefixer` + `browserslist` (piso Ch38) + fallbacks: clamp/min/overflow
  in-place, y `src/legacy-compat.css` (grid→flex + aspect-ratio→padding-bottom) TODO dentro de
  `@supports not(...)` → el moderno lo ignora. **Verificado moderno intacto en el build.**
- **Verificado por el usuario: móvil + desktop + TV moderna OK.** Pendiente: UH8500 mañana.
- **Tags de restauración** (local + GitHub): `antes-estilo-teles-antiguas` (`361a3e5`, revertir
  todo) y `compat-legacy-ok` (`4eaca8e`). Deps nuevas: plugin-legacy, terser, postcss-custom
  -properties, @csstools/postcss-global-data, autoprefixer (todas devDeps).

### 🛡️ SEGURIDAD — plan (audiencia: personal/familiar)
Riesgo #1 NO es XSS: la app es **pública** y usa el **RD PERSONAL** del usuario → cualquiera
que encuentre la URL lo quema (baneo cuenta RD / DMCA). Por eso, para uso personal:
- 🔴 **#1 Auth gate (password/PIN)** ~0.5 día — mismo patrón que RD_TOKEN: password en env var
  de Netlify; una function valida y emite token; `rd-token`/`rd-stream`/`rd-cleanup`/`subhash`
  exigen ese token o rechazan. Con esto el rate limiting deja de ser urgente.
- 🟢 **#2 Validación de URLs de streams** 2-4 hrs — barato, defensa en profundidad.
- ⬇️ Rate limiting + CSP → opcionales/después (CSP es frágil: puede romper iframes/imágenes TMDB).
- Estimación crítica real (auth + URLs): **~1 día**, no 3.

### 🐞 BUG ABIERTO — anime no reproduce en móvil
- **Síntoma:** en iPhone (Chrome/WebKit), reproducir un episodio de anime termina en
  error de JW Player *"This video file cannot be played (Error Code: 102630)"*.
  Confirmado por el usuario: **solo falla en móvil** — mismo navegador Chrome en
  desktop/TV sí reproduce (la diferencia real es el motor: WebKit obligatorio en
  iOS vs. Blink en desktop/TV).
- **Hipótesis sin confirmar:** los servidores default usan HLS (MSE), y Safari/WebKit
  iOS restringe MSE en iframes de terceros.
- **Fix intentado y DESCARTADO por el usuario** (commit `361a3e5`, sesión
  2026-06-30): preferir MP4Upload/no-HLS por defecto en móvil (`pickDefaultAnimeServerIdx`
  en `PlayerView.vue`) — **no resolvió el problema** ("no se reproduce").
  Investigación externa (WebSearch) quedó inconclusa por límite de herramienta, NO
  por evidencia que descarte la hipótesis.
- **Siguiente paso:** no volver a adivinar. Probar manualmente cada servidor del
  selector (StreamiX/HLS/MP4Upload — ya visible en móvil desde el fix de
  `ab0e921`) y reportar cuál reproduce, o retomar la investigación externa.
  Ver detalle completo en `sessions/2026-06-30.md §4`.

### 🎨 VISTA MÓVIL — resto del rediseño [COMPLETADO 2026-06-29/30]
Ya NO está pendiente el repaso general de móvil: se hizo un rediseño completo
estilo Netflix (hero, nav, detalle, fullscreen iOS, pills/categorías, flecha
volver, overflow, botones grandes) en dos sesiones. Detalle en
`sessions/2026-06-29.md` y `sessions/2026-06-30.md`. Sigue abierto SOLO el bug
de anime de arriba; el usuario puede seguir pidiendo ajustes puntuales al seguir
probando.

### ⏳ Otros pendientes (menor)
- **Layout TV:** el reproductor se corre hacia arriba dejando un espacio abajo (CSS, solo TV).
- Verificar en más navegaciones el fix del catálogo cortado en TV (ya confirmado "se ve mejor").

### ✅ Todo verificado en TV (sesión 2026-06-27)
- Seek fluido uniforme (`+400` cacheado). Badge de diagnóstico quitado. Catálogo ya no se corta
  al navegar (dispatch `resize` en TV).

### ✅ Sesión 2026-06-30 — Ajustes móvil (pills/nav/detalle) + intento anime [COMPLETADO parcial] (ver sessions/2026-06-30.md)
- Home móvil: pills de navegación + dropdown "Categorías" estilo Netflix, header "M Inicio".
- Fix crítico: selector de servidores de anime estaba OCULTO en móvil (el CSS de
  fullscreen lo tapaba junto al de RD) → ahora visible, el usuario puede cambiar de servidor a mano.
- Flecha volver en detalle, overflow horizontal clipeado, botones hero agrandados.
- Anime móvil: fix de default (MP4Upload/no-HLS) desplegado pero NO resolvió el bug (ver arriba).

### ✅ Sesión 2026-06-29 — Rediseño móvil estilo Netflix + fullscreen iOS [COMPLETADO] (ver sessions/2026-06-29.md)
- Hero/Nav/Detalle rediseñados a patrón Netflix (hero full-height, pills, nav 3 íconos).
- Fullscreen real en iOS vía `webkitEnterFullscreen()` (requestFullscreen de página no sirve en Safari).
- Subtítulos espejados a TextTrack nativo para que se vean en fullscreen nativo iOS.
- Fix audio fantasma en "continuar viendo" (Direct Play y pipeline `/t/` arrancan
  ya posicionados/muteados hasta completar el seek).

### ✅ Sesión 2026-06-27 — Panel episodios Netflix + RESUELTO seek en TV [COMPLETADO] (ver sessions/2026-06-27.md)
- **Panel de episodios estilo Netflix**: `EpisodePanel.vue` (acordeón, miniaturas TMDB,
  selector temporada), `NextEpisodeCard.vue` (card "Siguiente" en hover), `EpisodeTriggers.vue`
  (⏭/⧉ en barra de controles RD vía slot + mini-barra centrada en iframe/anime),
  `services/episodes.ts` (+10 tests). Reemplaza dropdowns/botón flotante viejos.
- **Navegación/fullscreen**: auto-hide uniforme; ⛶ unificado (flecha "Volver" visible);
  un solo ⛶ en móvil/tablet/desktop (RD); "Volver" respeta filtro de origen; fix double-back iframe.
- **⭐ RESUELTO seek lento SOLO en TV** (Cadena Perpetua): era un **EMPATE en el scoring**
  (`[RD+]` cacheado +200 = latino +200) → inglés-cacheado (`/t/` fluido) empataba con
  latino-no-cacheado (server-side lento) y Torrentio desempataba distinto por request → TV vs
  desktop daban resultados distintos. **FIX:** `[RD+]` cacheado **+200 → +400** (streamSelector.ts)
  → cacheado SIEMPRE gana → siempre `/t/` fluido, uniforme. + deshabilitada "preferencia latino
  server-side" (rdStream.ts) + `AbortSignal.timeout`→`AbortController` (rd-t-pipeline.ts, webOS) +
  quitado resolveCache (rd-tpipeline.js) + `no-store` index.html (public/_headers). **NO bajar el +400.**
- Diagnóstico: badge verde EN PANTALLA fotografiado en la TV (no hay consola en webOS).

### ✅ Sesión 2026-06-23 — Subtítulos /t/, prioridad latino, seek Direct Play, continuar viendo [COMPLETADO]
- **Subtítulos /t/**: `useSubtitles` ahora usa `timeOverride` (offset + currentTime) para sincronizar cues con pipeline /t/.
- **Scoring latino +200**: Dual-Lat sube al tope del ranking. Si el match en downloads no tiene latino pero hay Dual-Lat `[RD+]`, se resuelve server-side (solo LATINO, no castellano).
- **Fix seek Direct Play**: `seekOverride` ahora hace seek nativo con mute/unmute (audio no suena antes de imagen). `isTpipeline` se resetea al inicio de cada carga (antes quedaba true entre películas).
- **Toast "Seek fluido"**: indica al usuario si tiene Direct Play o /t/, y si el audio es latino.
- **Continuar viendo**: `positionSec` en ProgressEntry, posición real guardada cada 15s, restauración automática al reabrir, botón "Continuar 30:01".
- **Verificado en vivo**: The Thing (Direct Play), Wall-E (Dual-Lat /t/), El Padrino (/t/), Parásitos (Direct Play latino), La Momia (/t/ + subs).

### ✅ Sesión 2026-06-24 — Mejoras UI + restaurar posición /t/ [COMPLETADO]
- Círculo de score SVG quitado de tarjetas (poster limpio)
- Duración legible "2h 18min" en DetailView
- Botón "Continuar · 30 de 120 min" con contexto
- ContinueWatching y Mi Lista igualadas al catálogo (220px)
- Cache progresivo de runtime en localStorage
- Duración en MovieCard overlay (cuando hay cache)
- Restaurar posición en pipeline /t/ (tpipelineSeekTo)
- runtimeMin en ProgressEntry + duración en ContinueWatching

### ✅ Sesión 2026-06-24 (cont.) — Castellano sin bonus + series fix [COMPLETADO]
- Castellano ya no recibe bonus de idioma (+60/+90 → +5/+10 igual que inglés)
- Solo LATINO mantiene +200. Regla clara: Latino > Inglés+subs. Castellano = Inglés.
- FROM (serie): antes castellano 720p sin rdId. Ahora inglés 1080p H264 + pipeline /t/ + subs.
- Aplica a películas Y series (misma función scoreStream).

### ✅ Sesión 2026-06-24 (cont.) — Flujo anime LIMPIO [COMPLETADO]
**Decisión del usuario: anime SIEMPRE Anime1V (consistente, no híbrido).**
- Anime NUNCA pasa por RD (sourceIndex=0 directo al iframe).
- Una sola carga: HLS si existe, si no StreamiX (antes doble carga = flash + doble back).
- Selector consistente SUB/DUB/StreamiX/HLS, filtrado por idioma, HLS default. Latino en tab DUB.
- Volver navega DIRECTO al detalle (router.push, no router.back) → un click, sin pantalla blanca.
- closePlayer usa bumpGeneration → sin parpadeo del RD al volver.
- LECCIÓN: usar AskUserQuestion para decisiones de producto ambiguas (adiviné mal 3 veces).

### 🎯 PRÓXIMOS PASOS

#### 🎨 MEJORAR UI DEL REPRODUCTOR (material de estudio próxima sesión)
- **Barra inferior de episodios** (TEMP/EP selectors + botones "Anterior"/"Siguiente episodio"):
  se ven feos centrados abajo. Reubicar/rediseñar — quizás moverlos a otro lugar o estilo.
- **Botón flotante "Sig. Ep ›"** (esquina inferior derecha): mal ubicado, encima de los
  controles del iframe. Acomodar o integrar mejor.
- Revisar consistencia visual general del reproductor (anime iframe vs RD nativo).

#### 🐛 BUG PENDIENTE — flecha volver en fullscreen DESDE el reproductor
- El auto-hide de controles ("auto-hide controls" + "transparent activity-catcher overlay"
  para el iframe cross-origin) quedó funcionando: mover puntero → muestra, 3.5s quieto → oculta.
  Consistente para peli/serie/anime. Listener en `document` cubre fullscreen del video nativo.
- **FALTA**: cuando se hace fullscreen DESDE el reproductor (botón ⛶ del player), la **flecha
  volver** se esconde (lo demás funciona). Revisar z-index / posición de `.player-back` dentro
  del fullscreen nativo, o que el listener de actividad la revele en ese estado.
  Archivo: `PlayerView.vue` (`.player-back`, `armControlsHide`, `useFullscreen`).

#### Funcionalidad
1. **Cambio de audio en pipeline `/t/`** (Español ↔ Inglés).
2. **Auto-next en series**: verificar que funcione.
3. **Rooster Fighter no detectado como anime**: mejorar detección isAnime (géneros TMDB).
4. **Naruto (id 46260)**: se quedó en "Conectando con el servidor..." — revisar si Anime1V
   no tiene match o el iframe StreamiX no carga. Posible: fallback baseline no llegó.
5. **Probar pipeline `/t/` en TV** (LG webOS).
6. **Preview thumbnails al seekear** (requiere sprites).

### 🏆 Sesión 2026-06-16 (tarde/noche) — pipeline `/t/` de RD [DESCUBIERTO + VALIDADO EN DEMO → PORTADO 2026-06-21]
- **Pipeline `/t/` descubierto y PROBADO en la demo**: la API privada de RD
  (`{cdn}/t/{mediaId}/{hash}/{audio}/none/aac/full.mpd?t={seconds}`) genera DASH **en cualquier
  posición** → **far-seek universal** + **audio AAC siempre**. Confirmado con seeks reales a 30/48 min.
- Cómo obtener `mediaId`: GET `real-debrid.com/streaming-{rdId}?auth_token=` → regex `setMediaId("…")`.
  Luego `mediaInfos/{mediaId}` (cdn, hash, audio), `ping/{mediaId}/play`, `ping/{mediaId}/{seconds}`.
- ❌ **Fase 5 CANCELLED** — el pipeline `/t/` reemplaza al servidor ffmpeg propio.

### ✅ Sesión 2026-06-16 (mañana) — entregado y pusheado
- **Audio LATINO-only** (no castellano): si no hay latino → inglés+subs. El corredor ESP → inglés+subs.
- **Recuperación a posición** (no reinicia a 0 en pausa/stall).
- **Transcode liviano** (penaliza MKV pesados que RD no sostiene a tiempo real).
- ❌ **liveMP4 descartado** con evidencia (503 transcoding_error).
- ⚠️ Límite de transcode pesado RESUELTO por pipeline `/t/`.

### ✅ Sesión 2026-06-15 — entregado (reproductor robusto). OBJETIVO "que no se caiga" = CUMPLIDO.
- **Red anti-congelamiento**: far-seek en transcode YA NO mata la peli (vuelve solo). Afinado cercano/lejano.
- **Subtítulos humano>AI** (`scoreSubtitle` −8 AI): sub sincronizado. Offset persiste.
- **Híbrido idioma/seek** (`scoreStream`): Direct Play cuando hay MP4/H264/AAC cacheada; **latino** cuando solo hay transcode (Jack Ryan suena en latino). Inglés Direct Play no se sacrifica.
- **FASE 3 mediaInfos**: audio español por token real en transcode + audio "Surround" incompatible (Alien).
- ❌ **Pre-cacheo DESCARTADO** (`3e7c24c`): RD no baja títulos nuevos (0%), deja torrents muertos. Probado.

### 🔓 CONCLUSIÓN ACTUALIZADA: pipeline `/t/` resuelve el far-seek
El pipeline `/t/` de RD (`full.mpd?t={seconds}`) genera segmentos en cualquier posición →
**far-seek universal para cualquier archivo, sin servidor propio**. El límite de transcode
secuencial de RD ya no aplica cuando usamos la CDN privada.

### 🔬 Sesión 2026-06-16 (noche) — Demo `/t/` validada en `PRUEBAS/` [COMPLETADO ✅]
- **Servidor + frontend vanilla JS** probando todos los endpoints.
- **Far-seek confirmado**: 1s → 30min → 50min. Delay de 1-10s según caché.
- **Offset `_mpdOffset`**: Shaka muestra `currentTime=0` al cargar `?t=X` → display = offset + currentTime.
- **`waitForSegmentAt(t, 8000)`**: polling al CDN antes de cargar Shaka. Reduce Error 1003.
- **Freeze frame + loader + corte de audio**: UX fluida durante seek.
- **Barra custom 1px** azul/gris con tooltip, sin pelotita, 100% JS.
- **Backup**: `PRUEBAS_backup_20260616_2257/`.

### 🔥 PRÓXIMOS PASOS — Adaptar demo al proyecto principal
1. **Migrar `waitForSegmentAt` + offset + Shaka config** a `usePlayer.ts`
2. **Barra de progreso custom** como componente Vue (reemplazar `<video controls>`)
3. **Freeze frame + loader** en `TpipelinePlayer.vue`
4. **Corte de audio en seek** (pause+mute antes de `shakaPlayer.load()`)
5. **CDN mismatch**: usar `<BaseURL>` del MPD, no `mediaInfos.cdn`
6. **`full.mp4`** como toggle opt-in, no default

**✅ Sesión 2026-06-14 (reproductor) — entregado:** rediseño de scoring por tiers (P1 cacheado
→ P2 H264 → P3 AAC → P4 tamaño → P5 resolución → P6 idioma → P7 MP4-bonus); Direct Play vs
transcode; Shaka paciente; fix seek-muerte. Ver `sessions/2026-06-14.md` para el detalle.

**✅ Sesión 2026-06-13 (TV/webOS) — resuelto y confirmado por el usuario ("así quedó bien el flujo"):**
- Scrollbar horizontal azul al pie ("mini footer") — `body{overflow-x:hidden}` + ocultar scrollbar horizontal.
- Barra del navegador webOS tapaba el nav tras reproducir — `requestFullscreen` al "Volver" + **NavTV `position:sticky`** (sobrevive al `:fullscreen` de webkit, donde `fixed` se rompe).
- Tarjetas grises (skeleton) al scrollear/volver — skeleton transparente en TV + **cache SWR** en `useTmdbList` (memCache + localStorage TTL 6h).
- Nav saltaba a "Inicio" en la ficha — `lastSection` en `App.vue`.
- "Volver" de la ficha iba a Inicio — ahora `router.back()` → vuelve a la sección+género (commit 8cf3239).
- Detalle en `sessions/2026-06-13.md`.

**Pendientes inmediatos:**
1. **Afinar pipeline `/t/`**: hay que verificar que el CDN elegido en `mediaInfos` coincida con el del MPD, manejar errores de `mediaId` inválido/expirado, y hacer auto-fallback a transcode DASH legacy.
2. **Probar pipeline `/t/` en TV** (LG webOS) — el DASH nativo del TV puede tener limitaciones.
3. **Barra del navegador webOS:** funciona pero "falta afinarlo algo" (el usuario). Solución
   definitiva = empaquetar como **PWA / webOS .ipk** (sin chrome de navegador) — ya en backlog.
4. (Opcional) recordar posición de scroll de Películas al volver; clic en "Películas" recuerde el último género.

**Backlog (sin urgencia):** probar `rd-stream` end-to-end (título no cacheado); D-pad en tabs/
géneros (hoy solo puntero); capar grilla infinita; PWA; Anime/Deportes en Inicio.

**Arquitectura nueva:**
- Pipeline `/t/` es el método PRINCIPAL de streaming. Servicios: `rd-t-pipeline.ts`, `rd-media-infos.ts`, `useTpipeline.ts`, `TpipelinePlayer.vue`.
- Boot stage `rd-bootstrap` obtiene `mediaId` de la página de streaming de RD.
- ScoreStream tier P1b (pipeline `/t/` cuando `rdMediaId` existe, +20).
- Transcode DASH (`/streaming/transcode/{id}/dash.full`) es ahora **legacy/fallback**.
- ❌ Fase 5 (servidor ffmpeg propio) = **CANCELLED**.

**Recordatorios clave:** RD_TOKEN solo server-side; responder en español; no deploy/commit sin
permiso (salvo que el usuario lo pida); para foco/hover tv-mode usar `style.css` global (NO
`:global(html.tv-mode) .x:focus` en scoped → bug que aplica outline a `<html>`, ver coding-standards).

---

## ✅ Completado 2026-06-12 (sesión TV / Magic remote / deploy)

- **Deploy en vivo**: git + GitHub (`maldoxs/maldoxfilm-vue`) conectado a Netlify; push a `main` → auto-deploy. `RD_TOKEN` en env vars (usuario). `public/_redirects` → fin de los 404 en rutas/volver.
- **TV — pantalla azul / player parcial**: restaurada la regla `html.tv-mode .player-page { width:100vw; height:100vh }` que la migración había perdido (commit b3eebce). Era "te olvidaste de las resoluciones mayores".
- **TV — audit CSS del original**: tamaños de texto/foco D-pad; controles de hover (flechas, ✕) siempre visibles; card sin "salto" en hover.
- **Nav TV**: buscador/fullscreen a la derecha (`flex:1` en items), logo Oswald MALDOX(azul)/FILM(blanco), activo subrayado, buscar con texto.
- **Buscador TV**: agregado el input (faltaba); barra full-width; clic en toda la barra enfoca; Enter cierra el teclado.
- **Navegación espacial D-pad** (`useTvSpatialNav`): ←/→ card por card, ↓/↑ entre categorías, foco sigue al puntero; sin escaparse al nav (removido handler conflictivo de NavTV).
- **Guard anti-archivo-basura** (Scary Movie): descarte por nombre/tamaño + URL anulada. +6 tests.
- **Botón Reproducir**: 1 ícono ▶ + sentence case. **Mensaje** "Película no disponible" corto.
- **content-visibility removido** (efecto aparecer-al-scroll, a pedido del usuario).
- **ADR-006**: function `rd-cleanup` (limpieza de torrents).
- **Menú de géneros estilo Netflix (TV)** `TvGenreBar.vue`: activo anclado izquierda + borde blanco, deslizamiento, **scroll INFINITO** (lista triplicada, sin gap), filtro al asentarse. Solo TV.
- **Imágenes `eager`** (sin gris→foto) + **nav TV fondo negro sólido**.
- **"Línea azul" en TV eliminada** (commit d74e3be): bug de `:global(html.tv-mode) .x:focus` en scoped CSS que aplicaba un outline a todo el `<html>`. Reglas de foco movidas a `style.css` global con `:focus-visible`. **Diagnosticado inspeccionando el DOM en vivo** (preview puerto 5180 + CSSOM).
- **Audio/espacio-fantasma del player** (commit 5ba60e4): `usePlayer.destroy()` ahora hace `video.pause()`+vaciar src; `PlayerView` limpia el iframe → no sigue el audio ni queda footer fantasma al volver.
- Estado: **257 tests verdes**, `vue-tsc` limpio, build OK, todo pusheado a `main`.

## ⏳ Pendiente (sesión actual)
- **Nav principal se esconde "a veces" en TV** — falta clarificar en qué vista exacta (el usuario lo debe precisar). No reproducido.

---

## ✅ Completado

- **Reproductor RD operativo** en local (RD → DASH/HLS/HEVC directo).
- **Restauración de logs `[RD]` y `[SUB]`** (diagnóstico que se había perdido en la migración).
- **Backend de token RD**: Netlify Function `rd-token` + `.env` gitignored + `netlify.toml` + `vite.config` strictPort 5173.
- **Functions de subtítulos**: `subhash.cjs` + `offset.cjs` copiadas del original (+ `@netlify/blobs`).
- **Fix Shaka**: `new shaka.Player()` (era `new shaka()`).
- **Fix scroll**: quitado `scroll-snap-type: x mandatory` + `overflow-y: hidden` en carruseles (scroll vertical ya no se atrapa).
- **Inicio completo**: hero rotativo, tabs (Películas/Series/Populares), barra de géneros (13), Continuar viendo, ~11 carruseles, TOP 10 numerados estilo Netflix.
- **Películas/Series**: 5 secciones por género (🔥⭐🏆🎬🗓️) + lista completa de géneros.
- **Grilla "Ver todo"** (`/ver-todo`): scroll infinito (precarga + relleno encadenado), fondo "telón", pósters grandes, cabecera fija (overlay fullscreen). Género persistido en URL (`?genero=`) para que "Volver" mantenga el filtro.
- **Buscador**: pill colapsado → overlay centrado con × (estilo producción).
- **Nav**: logo Oswald (MALDOX azul + FILM blanco), links con pill en hover y azul sin subrayado en activo, fondo negro sólido. Fuente **Oswald** cargada (Google Fonts).
- **Auto-hide** del topbar del player (back + selector) tras 3.5s; flecha Volver se oculta también en iframe.
- **Player UI**: botones fullscreen reactivos (`:d` en vez de innerHTML), click en video = play/pausa, slider de volumen colapsable, offset de subtítulos dentro del panel ⚙️.
- **SCP creado y validado**: 19 docs, índice maestro en `project-map.md §0`, sin duplicidades (auditado), conectividad verificada.
- **Token RD actualizado y VERIFICADO** en `.env` — reproduce con `[RD] Match → id:` (DASH OK).
- **Arquitectura RD server-side (ADR-004)**: Netlify Function `rd-stream` (addMagnet→selectFiles→unrestrict→transcode DASH) para no-cacheado, sin Torrentio de puente, token oculto. Cableado inyectable (`serverResolve`), tests intactos. **Endurecida**: `pollInfo` con presupuesto global (espera conversión del magnet + instant-cache) → corrige el bug que impedía `ready:true`; devuelve `torrentId` para ADR-006. Cadena cliente verificada coherente. *Falta prueba end-to-end real con un título no cacheado.*
- **Subtítulos: decisión cerrada (ADR-005)** — RD no expone pistas (`getTextTracks()===[]`, sin endpoint en la doc) → se mantiene **OpenSubtitles**. Doc de referencia `realdebrid-api.md` + `CLAUDE.md` puente creados.
- **Optimización de memoria #1**: recuperado `content-visibility: auto` + `contain-intrinsic-size` en `Carousel`/`ContinueWatching`/`TopNumbered` → TV/móvil no se "pegan" en el Inicio. Imperceptible visualmente.
- **Optimización de memoria #2**: buffer de Shaka recortado en TV/móvil (`isLowMemoryDevice()` → `bufferBehind: 10`); desktop intacto.
- **Recuperación de stall a mitad de reproducción (DASH)**: monitor de avance de `currentTime`; si se traba ≥8s (típico: pausa larga → URL RD caducada) recarga el DASH desde la posición; tras 3 intentos sin éxito → cambia de fuente. (`usePlayer.ts`)

---

## 🔄 En progreso

- (vacío)

## ✅ Completado (cont. 3) — 2026-08-02

- **Fuga de torrents de Plan B/derivación /t/ — RESUELTA** (caso real: capturas de
  la cuenta RD del usuario mostrando "Doctor Sleep (2019) DC" + 3 archivos extra
  pegados para siempre). `resolveRawToRdId` (usado por la derivación de rdId null
  Y por cada alternativa que prueba el Plan B) agrega un torrent nuevo a la cuenta
  RD al seguir el link de Torrentio, pero ADR-006 solo cubría el torrent del
  camino legacy `rd-stream` — estos quedaban sin limpiar NUNCA. Fix: `rd-tpipeline.js`
  ahora detecta el torrent nuevo (snapshot de `/torrents` antes/después) y lo
  devuelve como `torrentId`; `usePlayer.ts` lo acumula en `sessionTorrentIds` (además
  del `currentServerTorrentId` legacy) y los borra todos juntos al cerrar/cambiar
  título vía el mismo `serverCleanup`/`rd-cleanup` de ADR-006.
- **Plan B ya no descarta alternativas para siempre tras un fallo transitorio —
  RESUELTA**. Bug real que explica el corte largo reportado ("esta se paro": buffer
  de 70s a 0 en ~12min, Plan B disparó una vez y nunca más): `tryPlanBSwap` usaba
  `tpipelineAlts.shift()` y descartaba la alternativa si `resolveRawToRdId` no
  devolvía id (típico: `/downloads` de RD aún no lista la entrada). Si esa era la
  única alternativa, `tpipelineAlts` quedaba vacío y el disparador preventivo
  (`tpipelineAlts.length` en la condición) dejaba de intentar por el resto de la
  sesión. Ahora las que fallan por esto (no por corte/duración distinta, que sigue
  siendo descarte permanente) vuelven a la cola al final de `tryPlanBSwap` para que
  un disparo posterior las reintente.
  Archivos: `netlify/functions/rd-tpipeline.js`, `src/services/rd-t-pipeline.ts`,
  `src/composables/usePlayer.ts`. 351/351 tests, `vue-tsc -b` OK, build OK.
  Commit `db05247`, deploy `6a6f7b49c42c54f374e0c019` (producción, live).
  **Pendiente de verificar con el usuario**: reproducir un título con alternativas
  Plan B y confirmar que (a) no aparecen torrents nuevos pegados en la cuenta RD
  tras cerrar, y (b) un stall sostenido dispara Plan B más de una vez si la primera
  alternativa falla transitoriamente.

## ✅ Completado (cont. 2)

- **ADR-006 — limpieza de torrents de RD IMPLEMENTADA**: function `rd-cleanup` (`DELETE /torrents/delete/{id}`) + `torrentId` propagado (rd-stream→bootstrap→rdStream→types→usePlayer) + borrado al cerrar/cambiar vía `navigator.sendBeacon` en `VideoPlayer.vue`. `serverCleanup` inyectable (default no-op), tests 251/251, build OK.

## ✅ Completado (cont.)

- **Caso "Weapons" / multi-audio sin rdId — RESUELTO** (2026-06-09): cuando el stream
  top no tiene `rdId` (DMCA/no cacheado), el "rescate" ahora recorre **todo el pool** y
  usa la **mejor versión cacheada** (con rdId) → entra a transcode DASH → audio + subs.
  Weapons ya reproduce con audio inglés + subtítulos español. Cambio contenido en
  `streamSelector.ts` → `resolveActiveStream` (solo corre donde antes daba `rdId null`;
  no afecta los casos que ya andaban). El error CORS `failed_infringement` que persiste
  es ruido esperado (intento fallido en la versión DMCA antes de pasar a la cacheada).

---

## ⏳ Pendiente

- **Afinar pipeline `/t/`**: elegir CDN correcta, manejar 403, detectar `mediaId` inválido, auto-fallback a transcode DASH legacy.
- **Probar pipeline `/t/` en TV** (LG webOS) — puede tener limitaciones de DASH nativo.
- **Optimización memoria #3**: capar la grilla infinita (`AllResultsView`) en TV/móvil.
- **Probar `rd-stream` end-to-end** con un título sin ninguna versión cacheada.
- Secciones del Inicio que faltan vs producción: **Anime** y **Deportes (EN VIVO)**.
- Las 2 filas **TOP numeradas** ya existen (Películas/Series); evaluar añadir más TOP si se requiere.
- Badge contador **"VER TODO 10K"** en carruseles (decorativo).
- Que el "Ver todo" del **Inicio** abra la grilla paginada (hoy va a `/peliculas` o `/series`).
- Completar los 20 requisitos originales (revisión final de paridad).

---

## ⚠ Riesgos

- **Token RD**: actualizado y verificado (2026-06-09, `[RD] Match` OK). Recordar: si caduca → `rdId null` → iframe (no es bug de código).
- **Functions solo con `netlify dev`** (no `npm run dev`): si se prueba con Vite puro, RD y subtítulos no funcionan.
- **Pipeline `/t/`**: el boot stage `rd-bootstrap` puede fallar si RD cambia la estructura de la streaming page o si el `mediaId` expira. Fallback a transcode DASH legacy.
- **CDN mismatch**: `mediaInfos.cdn` puede diferir del CDN real en el MPD; el navegador resuelve con `<BaseURL>` pero puede generar errores si el CDN del MPD está caído.
- **Acumulación de torrents en RD**: `rd-stream` hace `addMagnet` (escribe en la cuenta). Sin la limpieza de ADR-006 puede llegar al error 21. Mientras tanto, el camino server-side rara vez se ejecuta (la mayoría se resuelve en cacheado/rescate del pool). **2026-08-02**: se encontró y corrigió el mismo problema para `resolveRawToRdId` (derivación /t/ + Plan B) — ver "Completado (cont. 3)". La detección del torrent nuevo depende de un snapshot `/torrents` antes/después con `limit=15`; si dos resoluciones concurrentes agregan torrents casi al mismo tiempo, el diff podría fallar (no verificado en la práctica, caso raro: requiere Plan B disparando dos veces en paralelo, ya evitado por `planBSwapInFlight`).
- **`rd-stream` sin prueba end-to-end**: el `ready:true` real no se ha observado (solo error 35 en versión DMCA al probar el endpoint aislado).
