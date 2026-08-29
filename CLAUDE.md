# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

"God's Eye View" is a browser-based, real-time geospatial OSINT console: a CesiumJS + Google Photorealistic 3D Tiles globe overlaid with live public data layers (aircraft/ADS-B, ships/AIS, satellites/SGP4, earthquakes, wildfires, traffic, CCTV, radio stations, bikeshare, space launches, military installations from OSM) plus a voice-controlled AI copilot built on the OpenAI Realtime API. There is no backend framework — it's Vanilla JS + Vite.

The project has an explicit ethical boundary (from the README): it models events, assets, infrastructure, and systems — **not** named-person search, face recognition, or tracking individuals. PRs that cross that line won't be merged.

## Setup

- Node `>=24.14.0 <25 || >=26 <27` (enforced by `package.json`)
- `npm install`
- Copy `.env.example` → `.env` and set `GOOGLE_MAPS_API_KEY` (the only required key; needs the Map Tiles API enabled). Everything else in `.env.example` is optional and well-commented in place — don't re-derive it here.
- `npm run dev -- --host localhost --port 4173` (or `./scripts/dev-fresh.sh` on macOS, which also clears the Vite cache and pulls optional keys from Keychain)
- Open `http://localhost:4173`

## Common commands

- `npm run dev` — Vite dev server
- `npm run build` — production build → `dist/`
- `npm run preview` — serve the build (runs the same middleware/proxy stack as dev)
- `npm test` — full unit suite. `scripts/run-unit-tests.mjs` discovers all `src/**/*.test.mjs` (~154 files) and runs them via Node's built-in `node --test`, then runs two GC-bracketed allocation microbenchmarks serially (`--expose-gc --test-concurrency=1`); those two are only meaningful/enforced on Node 24 and are skipped on other supported engines.
- Run a single test file directly: `node --test src/data/flights.test.mjs` (any `*.test.mjs` path under `src/`)
- `npm run test:track` — regression/tracking invariants (`scripts/track-regression.mjs`). **Requires the dev server to already be running.**
- No linter or formatter is configured (no ESLint/Prettier, no lint script) — don't invent one.
- No CI — there's no `.github/` directory, so the pre-PR gate is manual: **`npm run build`, `npm test`, and `npm run test:track` must all stay green** before sending a PR.
- `scripts/qa-*.mjs` (~40 files) — headless Puppeteer QA harnesses for specific features (traffic, CCTV, floor-hold, voice routing, perf, etc.), run individually against a live dev server: `node scripts/qa-<name>.mjs`. Not part of the default test run.

## Architecture

**One process, two logical halves.** The browser client lives in `src/`, bootstrapped by `src/main.js`'s `init()`, which wires together the map stack, `DataLayerManager`, `StyleManager` (`src/ui.js`), `SceneDirector`, annotations, and voice. The "server" is not a separate app — it's Vite plugin middleware (`configureServer`/`configurePreviewServer`) living entirely inside `vite.config.js` (~7,400 lines), which runs identically under `vite dev` and `vite preview`. Every third-party secret stays server-side there; the browser only ever sees `GOOGLE_MAPS_API_KEY` and `CESIUM_ION_TOKEN` (both meant to be client-exposed and restricted at the provider) plus short-lived ephemeral tokens. Keep UI (`src/ui.js`) and layer logic (`src/data/<layer>.js`) separate, and route anything needing a private key through a `vite.config.js` proxy rather than fetching it with a raw key from the client.

**Data layer plugin pattern.** `DataLayerManager` (`src/data/manager.js`) is a registry that `src/main.js` populates via `dataManager.register(...)` for each of the ~13+ modules under `src/data/` (`flights.js`, `earthquakes.js`, `satellites.js`, `cctv.js`, `traffic.js`, `aisLiveVessels.js`, `militaryInstallations.js`, etc). Each module implements a common lifecycle contract — `init(viewer)`, `enable`/`disable`/`update`/`destroy`, `getStats()` — and `layerFeedState()` normalizes each layer's live status into a shared vocabulary (`nominal/loading/degraded/stale/fallback/unavailable`) that drives the HUD status chips. Model new data layers on an existing one rather than inventing a new shape.

**Voice/AI copilot — ephemeral-token WebRTC.** The client (`src/voice/gevRealtime.js`) requests a short-lived secret from same-origin `/api/realtime/token`; server-side, `vite.config.js` exchanges the real `OPENAI_API_KEY` for that ephemeral secret via OpenAI's realtime client-secrets endpoint and defines the tool schema as `GEV_REALTIME_TOOLS`. The browser then opens an `RTCPeerConnection` **directly to OpenAI** (bypassing the app's own server after the handshake), streams mic audio, and receives tool-call events over an `'oai-events'` data channel. `createGevActionRunner(...)` in `src/voice/gevActions.js` is the client-side dispatcher that executes each tool call against live app state (camera, `DataLayerManager`, `SceneDirector`, annotations) — new voice tools need a declaration in `GEV_REALTIME_TOOLS` (`vite.config.js`) plus an execution case here. Keep the tool surface tight and only confirm what actually happened. A separate, non-realtime call (`/api/openai/hud-summary`) generates the HUD's short scene-summary text.

**No database.** Persistence is filesystem-only: per-source disk caches under `.gev-cache/` (TTL logic lives inside each `vite.config.js` proxy plugin), `localStorage` for UI/panel/detection state, URL query params for shareable camera/style/layer state, and `.gev-logs/realtime-conversations.jsonl` for voice session debug logs.

**`docs/CURRENT-STATE.md`** is the authoritative, continuously-updated runtime-behavior reference — it's large, so search/read the relevant section rather than loading it wholesale. Read it before changing runtime behavior, and update it (plus `CHANGELOG.md`) in the same PR if behavior changes. Update `DATA_SOURCES.md` when adding or changing a data source (license/attribution) — never add data you don't have the right to redistribute.

## Coding style

ES modules, 2-space indent, single quotes, semicolons, JSDoc on exported/public functions. Match the surrounding code's idiom and comment density.
