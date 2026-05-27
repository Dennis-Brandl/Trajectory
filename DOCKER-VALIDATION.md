# Docker Validation Runbook — Trajectory Go-Public

**Purpose:** prove the Docker packaging for all five services actually builds and runs on a
machine **with Docker**, before any feature branch is merged to `main`. The Docker files were
authored on a machine without Docker, so this is the first real build.

**"Green" = all five containers build, start, stay up, serve their URL, and the Action Container
console can talk to the Action Container server.** Report results back and we proceed to merge.

> Static pre-checks already done (no Docker needed) and **passed**: every `COPY` source the builds
> need is committed (incl. `TrajectoryEditor/.npmrc`, the Actions `docker-entrypoint.sh`, the nginx
> template, the Python sidecar); shell scripts are committed with **LF** endings; Editor `/api/health`
> exists; `public/help.html` is committed in every app. So the likely *instant* failures are ruled
> out — what's left can only be confirmed by a real build.

---

## Prerequisites (on the Docker machine)

- **Docker** Engine or Desktop **with Compose v2** (the `docker compose` subcommand, not `docker-compose`).
- Internet access (pulls `node:20-alpine`, `node:20.19-alpine`, `node:22-alpine`, `nginx:alpine` and runs several `npm ci`/`npm install`).
- Free TCP ports **3000, 3001, 3002, 3003, 3004** on the host.
- A few GB of disk for images/layers. First build is slow (multiple npm installs + a native `better-sqlite3` compile + two Vite builds) — **budget ~5–15 min**.

Commands below are bash/WSL/macOS style. On Windows PowerShell, use `curl.exe` instead of `curl`, and `$env:VAR` for env vars.

---

## Step 1 — Get the exact pinned code

Clone the umbrella and move the submodules to the `feat/docker-setup` pins (a plain checkout does **not** move submodules — the `submodule update` is required):

```bash
git clone https://github.com/Dennis-Brandl/Trajectory.git
cd Trajectory
git checkout feat/docker-setup
git submodule update --init --recursive
```

Confirm the submodules landed on the exact go-public commits (detached HEAD is expected and fine):

```bash
git -C TrajectoryEditor        rev-parse --short HEAD   # expect 2b824bf
git -C TrajectoryRuntime       rev-parse --short HEAD   # expect 33e96bb
git -C TrajectoryActions       rev-parse --short HEAD   # expect 5b5724e
git -C TrajectoryActionTester  rev-parse --short HEAD   # expect c5d8815
```

| Submodule | Branch (pin source) | Expected HEAD |
|---|---|---|
| TrajectoryEditor | `feat/help-file` | `2b824bf` |
| TrajectoryRuntime | `feat/rest-and-file-changes` | `33e96bb` |
| TrajectoryActions | `feat/docker-setup` | `5b5724e` |
| TrajectoryActionTester | `feat/docker-setup` | `c5d8815` |
| (umbrella `Trajectory`) | `feat/docker-setup` | `c6df4be` |

> The 5th submodule, **TrajectoryManager** (on `main`), is **not** part of go-public and is **not** a
> compose service — it's harmless. If you want to skip cloning it, init only the four you need:
> `git submodule update --init TrajectoryEditor TrajectoryRuntime TrajectoryActions TrajectoryActionTester`.

---

## Step 2 — Build and start everything

From the umbrella root (the directory with `docker-compose.yml`):

```bash
docker compose up --build -d      # build all 5, start detached
docker compose ps                 # all 5 should be "running"; editor/actions-server show health
docker compose logs -f            # watch; Ctrl-C to stop following (containers keep running)
```

Expected services and URLs:

| Service | Compose name | URL |
|---|---|---|
| Editor (authoring) | `editor` | http://localhost:3000 |
| Runtime (execution UI) | `runtime` | http://localhost:3001 |
| Action Container — server (REST) | `actions-server` | http://localhost:3002 |
| Action Container — console (UI) | `actions-console` | http://localhost:3003 |
| Action Tester | `tester` | http://localhost:3004 |

If a service exits or `ps` shows it unhealthy/restarting, jump to **Troubleshooting**.

---

## Step 3 — Per-service functional checks

**Editor — http://localhost:3000**
- [ ] App UI loads.
- [ ] `curl -sf http://localhost:3000/api/health` returns JSON (DB connectivity OK) → `docker compose ps` shows `editor` **healthy**.
- [ ] http://localhost:3000/help.html renders the in-app Help guide (with screenshots). *(This is the bug-fix being validated — the image must ship `help.html`.)*
- [ ] Data persists: it writes SQLite to the `trajectoryeditor-data` volume.

**Runtime — http://localhost:3001**
- [ ] Web UI loads (static nginx).
- [ ] **Settings → About → "Open Help Guide"** opens `/help.html`.

**Action Container server — http://localhost:3002**
- [ ] `docker compose ps` shows `actions-server` **healthy** (TCP probe on 3002 — there is no public `/health` route yet).
- [ ] `docker compose logs actions-server` shows it listening on 3002 with no crash loop (watch for native-module or Python-sidecar errors).

**Action Container console — http://localhost:3003**  ← the cross-repo build; the main thing this whole exercise validates
- [ ] Console UI loads.
- [ ] **Help** link in the top nav opens `/help.html`.
- [ ] The console shows **server-backed data without connection errors** (lists/loads from the server). This proves the nginx reverse-proxy of `/trajectory` + `/management` to `actions-server:3002` works (SSE-safe), end-to-end across the two images.

**Action Tester — http://localhost:3004**
- [ ] App loads (single-file build).
- [ ] **Help** link opens `help.html` (relative companion file).
- [ ] Point it at `http://localhost:3002` and exercise an action against the Action Container server (server CORS is `*`).

---

## Step 4 — Why the Action Container is the risk

`editor`, `runtime`, `tester` each build **only from their own repo** — low risk, self-contained.

Both Action Container images build with the **umbrella root as context** (`context: .`,
`dockerfile: TrajectoryActions/Dockerfile`) because the console imports `@trajectory/ui` and
`@trajectory/tokens/dist/*.css` from **TrajectoryEditor** via `file:` links. The builder reproduces
the dev layout: installs the Editor workspace, builds tokens, then installs + builds Actions, then
esbuild-bundles the server. The root `.dockerignore` whitelists only `TrajectoryActions` +
`TrajectoryEditor` so this context stays small. **If anything fails, it's most likely here** — see
the fallbacks below.

---

## Troubleshooting (symptom → cause → fix)

| Symptom | Likely cause | Fix |
|---|---|---|
| `npm ci` fails with `EUSAGE` / lockfile-not-in-sync (Editor **or** Actions stage) | package-lock drift | In `TrajectoryActions/Dockerfile`, change the relevant `npm ci` to `npm install`. Lines: **44** (`cd TrajectoryEditor && npm ci`) and **50** (`cd TrajectoryActions && npm ci`). |
| `COPY ... not found` during Actions build | file excluded by context/whitelist | Confirm you're building from the **umbrella root** (not inside `TrajectoryActions`); check root `.dockerignore` re-includes `!TrajectoryActions` and `!TrajectoryEditor`. |
| Console build fails resolving `@trajectory/ui` or `@trajectory/tokens/dist/*.css` | tokens not built / Editor packages missing | Ensure the tokens build step ran (`npm -w @trajectory/tokens run build`, Dockerfile line **46**) and `TrajectoryEditor/packages` is present in context. |
| `better-sqlite3` / `node-gyp` build error | native module compile | Builder already installs `python3 make g++`; if it still fails, check the Node base image and that the toolchain layer ran. |
| `actions-console` returns 502 / "Bad Gateway" | server not up yet or wrong upstream | `actions-console` `depends_on: actions-server`; check `actions-server` logs; env `ACTIONS_API_UPSTREAM=http://actions-server:3002` (set by root compose). |
| `actions-server` "unhealthy" but logs look fine | it's only a TCP probe | Confirm it serves; consider adding an unauthenticated `/health` route as a follow-up (would allow an HTTP healthcheck). Not a blocker. |
| `docker-entrypoint.sh: not found` / `no such file` (Editor or Actions) | CRLF line endings | Should not happen (committed as LF), but if it does: re-checkout with LF / run `dos2unix` on the script. |
| `editor` unhealthy | `/api/health` failing or `/data` perms | Check `editor` logs; the entrypoint prepares `/data` for the non-root `appuser`. |
| Python sidecar errors at action runtime (Actions) | `python` not resolvable | The server image symlinks `python`→`python3`; verify it exists in the running container. |
| Port already allocated | host port in use | Stop the conflicting process or edit the `ports:` host side in `docker-compose.yml`. |

---

## Optional — build each app alone (to isolate a failure)

```bash
docker compose up --build editor          # 3000  (or: cd TrajectoryEditor       && docker compose up --build)
docker compose up --build runtime         # 3001  (or: cd TrajectoryRuntime      && docker compose up --build)
docker compose up --build tester          # 3004  (or: cd TrajectoryActionTester && docker compose up --build)
# Action Container (needs the Editor sibling — present in the umbrella clone):
cd TrajectoryActions && docker compose up --build     # server 3002 + console 3003
```

---

## Teardown

```bash
docker compose down       # stop + remove containers/network (keeps data volumes)
docker compose down -v    # also delete the SQLite volumes (trajectoryeditor-data, trajectoryactions-data)
```

---

## Success criteria (from the design spec) — report these back

1. With Editor + Actions as siblings, the Action Container **server (3002) + console (3003)** come up and the console reaches the server.
2. Each self-contained app's own build works: **Editor 3000 (incl. the Help guide), Runtime 3001, Tester 3004.**
3. The root `docker-compose.yml` brings up **all five**.
4. `npm install` + documented commands still run every app locally (see `DOCKER-README.md`).
5. No internal codenames remain; the Editor image serves `/help.html`.

**When you've run this, tell me:** which services came up, any errors/log excerpts (especially the
Actions console build), and whether you had to apply any fallback. If it's green, we merge the
feature branches to `main` (Editor/Actions/Tester are clean fast-forwards; Runtime gets a 2-commit
cherry-pick onto fresh `main`; then re-pin the umbrella submodules).
