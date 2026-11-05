# State Management — MaldoxFilm (Pinia)

> `src/stores/`. 4 stores. Persistencia en localStorage vía `services/safeStorage.ts`.

---

## `player.ts` — `usePlayerStore`

Estado del reproductor (reemplaza la global `playerState` del original).

```ts
state: {
  current: PlayerState,   // {id, type, season, episode, totals, title, runtimeMin, isAnime, animeTitle, animeTmdbId}
  generation: number,     // contador para cancelar async stale
  sourceIndex: number,    // RD_SRC_IDX=99 por defecto; 0=UnlimPlay, 1=vidlink
  isActive: boolean,
}
```

Acciones clave:
- `load(params)` → incrementa `generation`, **resetea `sourceIndex = RD_SRC_IDX`**
  (clave: un título nuevo arranca en RD, no arrastra la fuente del anterior), fija `current`.
- `bumpGeneration()` → renueva generación en cada `loadActiveSource` (cambio de ep/fuente) sin pisar `current`.
- `setSourceIndex(idx)`, `setEpisode(s,e)`, `setTotals(...)`, `presetAnimeDetection(...)`, `close()`.
- Getter `isStale(myGen)` → `generation !== myGen`.

---

## `progress.ts` — `useProgressStore`

"Continuar viendo". localStorage `sx_prog_v2`.

```ts
state: { all: ProgressMap }  // Record<"type:id", {id,type,pct,season?,episode?,title?,ts}>
```
Acciones: `get(id,type)`, `save(id,type,data)` (mergea + `ts:Date.now()` + persiste),
`remove(id,type)`, `clearAll()`.

---

## `myList.ts` — `useMyListStore`

Mi Lista. localStorage `sx_mylist`. Formato MIXTO preservado (strings legados + objetos).

---

## `device.ts` — `useDeviceStore`

Modo de dispositivo (`tv`/`desktop`/`mobile`), `isTV`, `isMobile`. `detect()` corre en
`main.ts` antes de montar.

---

## Convenciones

- Estado de UI/datos → store (`ref`/reactive).
- Punteros a instancias de terceros (Shaka, hls.js) → variables de módulo en el
  composable, NO en stores.
- Persistencia siempre vía `safeStorage` (wrapper seguro ante `localStorage` no disponible).
