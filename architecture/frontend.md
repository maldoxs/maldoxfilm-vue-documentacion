# Frontend — MaldoxFilm (Vue 3)

> Arquitectura de la capa Vue.

---

## Stack

- **Vue 3.5** Composition API, `<script setup lang="ts">`.
- **Vite 8** (dev/build), **TypeScript ~6**, **vue-tsc** (type-check + build `-b`).
- **Pinia 3** (estado), **Vue Router 4** (rutas).
- **Vitest 4** (tests de servicios puros).

---

## Capas y responsabilidades

```
views/        páginas (rutas). Arman composables + componentes.
components/   UI declarativa (props in, eventos out). Scoped CSS.
composables/  orquestación con efectos (DOM, red, timers, MediaSource).
services/     lógica PURA + clients de red. Testeada.
stores/       estado reactivo Pinia + persistencia localStorage.
```

Regla: **la decisión ("qué") vive en `services/` puros y testeados**; el composable
solo aplica el resultado sobre el mundo real (DOM/red).

---

## Bootstrap (`main.ts`)

Orden intencional:
1. `createApp(App)` + `createPinia()`.
2. `deviceStore.detect()` ANTES de montar (evita "flash" de layout incorrecto).
3. `await createAppServices({isTvMode})` — arma clientes de red + resuelve RD token.
4. `app.provide(APP_SERVICES_KEY, services)`.
5. `app.use(router)` + `app.mount('#app')`.

`createAppServices` es **async** porque `fetchRdToken()` pide el token a la Netlify
Function. Se espera antes de montar para que el reproductor ya tenga el token.

---

## Inyección de dependencias

- Clientes de red (TMDB/Torrentio/RD + `RdStreamResolver`) se crean UNA vez en
  `services/bootstrap.ts` y se proveen con `app.provide(APP_SERVICES_KEY)`.
- Componentes/composables los consumen con `useAppServices()`
  (`inject(APP_SERVICES_KEY)`).
- **No** instanciar clientes sueltos.

---

## Estilos

- `<style scoped>` por componente.
- Variables CSS globales en `src/style.css`.
- Fuentes Oswald + Roboto cargadas en `index.html` (Google Fonts).
- Padding horizontal estándar de secciones/carruseles: **52px**.
- Carruseles: `overflow-y: hidden` obligatorio (evita atrapar el scroll vertical).

---

## Layout responsive

`App.vue` renderiza el nav según `deviceStore.mode`:
- `desktop` → `NavDesktop` (barra superior).
- `mobile` → `NavMobile` (nav inferior).
- `tv` → `NavTV` (sidebar/topnav, pósters/paddings mayores).

`<router-view>` dentro de `<main class="app-main">`.

---

## Patrón de cancelación async (reproductor)

`playerStore.generation` se incrementa en cada carga. Los callbacks async capturan
`myGen` al iniciar y chequean `playerStore.isStale(myGen)` antes de tocar el DOM
(equivale al `if(_playerGen !== myGen) return;` del original).
