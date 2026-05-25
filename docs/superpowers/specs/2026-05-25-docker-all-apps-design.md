# Docker packaging for all four Trajectory apps — Design (standalone tree)

**Date:** 2026-05-25
**Status:** Approved; implementing in the standalone tree.
**Supersedes:** the umbrella-based draft in `C:\Trajectory\Trajectory\docs\…` (that tree is stale — see memory `workspace-canonical-standalone`).

## Goal

After a public clone, every app runs two ways: **(a) Docker** (`docker compose up --build`) and **(b) local rebuild** (`npm install` + dev/build). The Action Container **console ships with the server**.

## Canonical layout & the one structural constraint

Work happens in the standalone sibling clones under `C:\Trajectory\`:
`TrajectoryEditor` (branch `feat/help-file`), `TrajectoryRuntime` (`feat/rest-and-file-changes`), `TrajectoryActions` (`main`), `TrajectoryActionTester` (`main`). `C:\Trajectory\` itself is **not** a git repo.

Three apps build entirely from their own repo. **Only TrajectoryActions** (server *and* console) needs **TrajectoryEditor present as a sibling** at build time — the console imports `@trajectory/{ui,tokens}` via `file:../../../TrajectoryEditor/packages/*`, and the workspace `npm ci` resolves it. Resolution chosen: **parent-directory build context** (the dir holding both repos = `C:\Trajectory\` now, a rebuilt umbrella later — same relative layout, same Dockerfile).

## Port map (Docker)

`Editor 3000 · Runtime 3001 · Action Container server 3002 · console 3003 · Action Tester 3004`

## Per-app deliverables

### TrajectoryEditor (exists — modify)
- Rename "Trajectory MD" → "Trajectory Editor" in Dockerfile/compose/DOCKER-README.
- **Bug fix:** Dockerfile builds the frontend with `RUN npx vite build`, which **skips `build:help`**, so the image ships without `public/help.html` (toolbar Help → 404). Change to generate it: `RUN npm -w @trajectory/tokens run build && npm run build:help && npx vite build` (build:help reads `HELP.md`→`public/help.html`; Vite copies `public/`→`dist/`→ served at `/help.html`).
- Add a "Rebuild locally" section to DOCKER-README.

### TrajectoryRuntime (exists — modify)
- Rename "Trajectory RT" → "Trajectory Runtime" everywhere.
- Add a "Rebuild locally" section. Update the "Running Both…" section to point at the new root compose. (nginx static; no entrypoint/db.)

### TrajectoryActionTester (create — self-contained)
Multi-stage: `node:20-alpine` `npm ci`/`install` + `npm run build` (vite-plugin-singlefile → `dist/index.html`) → `nginx:alpine` serves `dist/`. SPA fallback. Port 3004:80. No proxy (user types the Action Container URL; server CORS is `*`). Files: `Dockerfile`, `.dockerignore`, `docker-compose.yml`, `DOCKER-README.md`.

### TrajectoryActions (create — built from parent context)
One multi-stage `Dockerfile` at the Actions repo root, built with the **parent dir** as context (so `TrajectoryEditor/packages/{tokens,ui}` are reachable):
- **`builder`** (`node:20-alpine` + `python3 make g++`): COPY `TrajectoryEditor/packages/{tokens,ui}` + `TrajectoryActions` preserving the `../../../` layout; `npm ci`; build tokens; `tsc --build` (storage/engine/server → dist); `vite build` (console → dist).
- **`server`** target (`node:20-alpine` + `python3` + a `python`→`python3` shim, because the engine spawns the sidecar as `python`): copy compiled dist + `node_modules` (better-sqlite3 native already built on the same base) + `packages/python-sidecar` (stdlib-only); non-root; `/data` volume; `PORT=3002`, `DB_PATH=/data/trajectory.db`, `SIDECAR_SCRIPT=/app/packages/python-sidecar/sandbox_runner.py`; entrypoint ensures `/data`; healthcheck on an unauthenticated `/management/v1` route (confirm during impl; that prefix is not behind the API-key guard) or a TCP check; `node packages/server/dist/index.js`.
- **`console`** target (`nginx:alpine`): serve SPA + reverse-proxy `/trajectory/` and `/management/` to the server, SSE-safe (`proxy_buffering off`, long read timeout, HTTP/1.1). Upstream from an nginx `templates/*.template` envsubst var (`${ACTIONS_API_UPSTREAM}`), so one image works as service `server` (Actions compose) or `actions-server` (root compose).

`TrajectoryActions/docker-compose.yml`: `server` (target `server`) + `console` (target `console`, `depends_on` server, `ACTIONS_API_UPSTREAM=http://server:3002`), both `build.context: ..`, `/data` volume. Run from inside the Actions repo (Editor present as sibling). Console `package.json` unchanged. Files: `Dockerfile`, `docker-entrypoint.sh`, `apps/console/docker/default.conf.template`, `.dockerignore`, `docker-compose.yml`, `DOCKER-README.md`.

### Root orchestration (create at `C:\Trajectory\`)
- `docker-compose.yml` — runs all five services. Editor/Runtime/Tester build with `context: ./Trajectory<App>`; the Actions `server`+`console` build with `context: .` (parent) + `dockerfile: TrajectoryActions/Dockerfile` + `target:` , service named `actions-server` (and `ACTIONS_API_UPSTREAM=http://actions-server:3002`).
- `.dockerignore` — **whitelist** style (`*` then `!TrajectoryActions`, `!TrajectoryEditor/packages`, plus needed Editor workspace files) so the Actions build context stays small despite `C:\Trajectory\` holding unrelated dirs (stale `Trajectory/` umbrella, `TrajectoryManager/`, `docs/`, `migration-plan/`, `.pytest_cache/`).
- `DOCKER-README.md` — run-all-with-Docker + run-all-locally; per-app port table.

## Naming rename map
"Trajectory MD" → **Trajectory Editor**; "Trajectory RT" → **Trajectory Runtime**; new: **Trajectory Action Container** (Actions), **Trajectory Action Tester** (Tester).

## Out of scope
Publishing shared packages; TrajectoryManager; CI/registries/TLS/k8s; rebuilding the umbrella (later); making a lone Actions clone build without an Editor sibling.

## Success criteria
1. With Editor+Actions as siblings, `docker compose up --build` in `TrajectoryActions` brings up server (3002) + console (3003), and the console reaches the server.
2. Each self-contained app's own `docker compose up --build` works (Editor 3000 incl. the Help guide, Runtime 3001, Tester 3004).
3. Root `C:\Trajectory\docker-compose.yml` brings up all five.
4. `npm install` + documented commands still run every app locally.
5. No internal codenames remain; Editor image serves `/help.html`.
