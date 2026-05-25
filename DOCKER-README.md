# Trajectory Platform — Docker Deployment Guide

Run the whole Trajectory platform — **Editor**, **Runtime**, **Action Container** (server + console), and **Action Tester** — with one command, or each app on its own. Every app can also be rebuilt locally without Docker.

## Layout

This guide lives in the directory that holds the app repos as **siblings**:

```
<this directory>/
  docker-compose.yml      <- run everything from here
  .dockerignore
  TrajectoryEditor/
  TrajectoryRuntime/
  TrajectoryActions/
  TrajectoryActionTester/
```

The Action Container console depends on the Editor's shared UI library, so the Action Container images build with **this directory** as their context (the Editor must be present as a sibling). The other three apps build from their own subdirectories.

## Run everything with Docker

### Prerequisites
- [Docker](https://docs.docker.com/get-docker/) (Engine or Desktop) with Compose

### Quick Start
```bash
docker compose up --build -d
```

| App | URL |
|---|---|
| Editor (workflow authoring) | http://localhost:3000 |
| Runtime (execution web UI) | http://localhost:3001 |
| Action Container — server (REST) | http://localhost:3002 |
| Action Container — console (UI) | http://localhost:3003 |
| Action Tester | http://localhost:3004 |

### Common Commands
```bash
docker compose up --build -d     # build + start all
docker compose up -d editor      # start just one service
docker compose down              # stop all
docker compose down -v           # stop + delete the SQLite volumes
docker compose logs -f           # live logs
docker compose ps                # status
```

Per-app details (config, data, single-app runs) are in each repo's `DOCKER-README.md`.

## Run everything locally (without Docker)

Each app runs from source with Node.js 20+ (the Action Container also needs Python 3 on PATH as `python`). Open separate terminals:

```bash
# Editor — API :3001 + UI :5174  (run build:help once for the in-app Help guide)
cd TrajectoryEditor   && npm install && npm run build:help && npm run dev:all

# Runtime — UI :5173
cd TrajectoryRuntime/engines/web-ui && npm install && npm run dev

# Action Container — server :3002 + console :5176  (needs TrajectoryEditor as a sibling)
cd TrajectoryActions  && npm install && npm run dev

# Action Tester — UI on a printed port
cd TrajectoryActionTester && npm install && npm run dev
```

## Validation note

These Docker files were authored on a machine without Docker, so they have not been built. Validate with a real `docker compose up --build` before publishing. The lowest-risk apps are Runtime and Tester (self-contained static builds); the Editor adds the in-app Help guide; the **Action Container console** — which builds the Editor's UI library from source across repos — is the piece most worth confirming first.
