# W1 — Publish Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make all four Trajectory repos safe and professional to publish as a public demonstration release — remove the leaked secret/PII, standardize on Apache-2.0/© Dennis Brandl, purge internal-only material + junk + dead code from the public drop, and add real READMEs — without changing runtime behavior.

**Architecture:** One coordinated wave delivered as **four independent per-repo PRs** off each repo's `origin/main` on a `hardening/release` branch. Cross-repo transformations (license-header rewrite, ignore policy, README template) share assets defined once in §0. The only behavioral code in W1 is **adding regression tests** that lock the Editor's already-correct user-management authorization; everything else is deletion, config, license text, and docs.

**Tech Stack:** Node 22 monorepos (npm workspaces), Vite/React, Hono server (Editor), Drizzle/better-sqlite3, Kotlin (Runtime engines), vitest. Source review: `C:\Trajectory\_release-review-2026-06-04\RELEASE-READINESS-REPORT.md`. Spec: `docs/superpowers/specs/2026-06-04-trajectory-demo-hardening-design.md`.

---

## Conventions (apply to every task)

- **Commit trailer (REQUIRED on every commit):** end each `git commit` with a second `-m`:
  `-m "Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"`
  Shown explicitly in Task A0; thereafter written as `# +trailer` for brevity but still REQUIRED.
- **Read-only outside the target repo.** Never push or open a PR without the user; "open a PR" steps stop for the user (sequential checkpoints).
- **Verify before commit.** Each task's verification command must pass (shown with expected output) before the commit step.
- Run all commands from the repo root unless a `cd` is shown.

---

## §0 Shared assets (create once, before the repo phases)

### 0.1 License header strings (exact)

OLD block (currently in source files — two consecutive lines):
```
// Copyright (c) 2026 Saturnis.io. All rights reserved.
// Licensed under the GNU AGPL v3. See LICENSE.md for details.
```
NEW block (replacement — two lines):
```
// Copyright (c) 2026 Dennis Brandl
// Licensed under the Apache License, Version 2.0. See LICENSE for details.
```

### 0.2 One-shot relicense script

- [ ] **Step 1: Create the shared script** `C:/Trajectory/_release-review-2026-06-04/tools/relicense-headers.mjs`

```js
// relicense-headers.mjs — one-shot: AGPL/Saturnis header -> Apache/Dennis Brandl.
// Also fixes the dead "See LICENSE.md" reference (new line points at LICENSE).
// Usage: node relicense-headers.mjs <repoRoot> [--dry]
import { readFileSync, writeFileSync, readdirSync, statSync } from 'node:fs'
import { join, extname } from 'node:path'

const root = process.argv[2]
const dry = process.argv.includes('--dry')
if (!root) { console.error('usage: node relicense-headers.mjs <repoRoot> [--dry]'); process.exit(1) }

const OLD1 = '// Copyright (c) 2026 Saturnis.io. All rights reserved.'
const NEW1 = '// Copyright (c) 2026 Dennis Brandl'
const OLD2 = '// Licensed under the GNU AGPL v3. See LICENSE.md for details.'
const NEW2 = '// Licensed under the Apache License, Version 2.0. See LICENSE for details.'

const EXDIRS = new Set(['node_modules','dist','build','.git','.planning','docs','coverage','.gradle','.worktrees','.superpowers','out','.next','kotlin-js-store'])
const EXT = new Set(['.ts','.tsx','.js','.jsx','.mjs','.cjs','.kt','.kts'])

let scanned = 0, changed = 0
function walk(dir) {
  for (const name of readdirSync(dir)) {
    const p = join(dir, name)
    let st; try { st = statSync(p) } catch { continue }
    if (st.isDirectory()) { if (!EXDIRS.has(name)) walk(p); continue }
    if (!EXT.has(extname(name))) continue
    scanned++
    const orig = readFileSync(p, 'utf8')
    if (!orig.includes(OLD1) && !orig.includes(OLD2)) continue
    const next = orig.split(OLD1).join(NEW1).split(OLD2).join(NEW2)
    if (next !== orig) { changed++; if (!dry) writeFileSync(p, next); console.log(`${dry ? '[dry] ' : ''}updated ${p}`) }
  }
}
walk(root)
console.log(`scanned ${scanned} source files; ${changed}${dry ? ' would be' : ''} updated`)
```

- [ ] **Step 2: Smoke-test in dry mode against the Editor**

Run: `node C:/Trajectory/_release-review-2026-06-04/tools/relicense-headers.mjs C:/Trajectory/TrajectoryEditor --dry`
Expected: prints `... updated ...` lines and a final `scanned NNN source files; 529 would be updated` (≈529; the exact count may differ slightly — the point is it finds the headers and writes nothing in dry mode).

*(No commit — this tool lives outside the repos.)*

### 0.3 Ignore-policy block (appended to each repo's `.gitignore`)

```
# --- Public-release hygiene (internal-only; not shipped) ---
.planning/
.claude/
tasks/
docs/superpowers/
*.docx
*.pptx
*.continue-here.md
```
Plus a per-repo line to stop tracking generated help (path differs per repo — given in each phase).

---

## Phase A — TrajectoryEditor PR  (`C:/Trajectory/TrajectoryEditor`)

Biggest PR: carries the secret/PII deletion + the authz regression tests.

### Task A0: Branch + verify baseline green

**Files:** none (git + test).

- [ ] **Step 1: Create the hardening branch off origin/main**

```bash
cd C:/Trajectory/TrajectoryEditor
git fetch origin
git switch -c hardening/release origin/main   # fallback if no origin/main: git switch -c hardening/release main
git status
```
Expected: `On branch hardening/release`. (If `origin/main` is missing, confirm the correct base with the user before proceeding.)

- [ ] **Step 2: Confirm the suite test baseline passes** (so later failures are attributable)

Run: `npm run test:run -- src/server/routes/__tests__/users.test.ts`
Expected: PASS (all existing user-route tests green).

### Task A1: Lock user-management authorization with regression tests (TDD)

The authz already exists (`src/server/routes/users.ts` checks `isAdministrator !== true → 403`), but `users.test.ts` only tests the **401-unauthenticated** path, not the **403-authenticated-non-admin** path. Add the missing regression tests so the fix can't silently regress. Mirror the pattern in `src/server/routes/__tests__/backup.test.ts:342-348`.

**Files:**
- Test: `src/server/routes/__tests__/users.test.ts` (add 3 cases)

- [ ] **Step 1: Add the failing-then-passing regression tests**

Add these three `it(...)` cases inside the existing top-level `describe` (reuse the file's existing `insertUser`, `mockSession`, `request` helpers):

```ts
it('GET /api/users returns 403 for an authenticated non-admin', async () => {
  const regular = insertUser('user-1', 'regular@example.com', { isAdministrator: false })
  mockSession(regular)
  const res = await request('GET', '/api/users')
  expect(res.status).toBe(403)
  const body = (await res.json()) as { error: string }
  expect(body.error).toBe('Admin access required')
})

it('PATCH /api/users/:id returns 403 for an authenticated non-admin', async () => {
  const regular = insertUser('user-1', 'regular@example.com', { isAdministrator: false })
  const victim = insertUser('user-2', 'victim@example.com', { isAdministrator: false })
  mockSession(regular)
  const res = await request('PATCH', `/api/users/${victim.id}`, { isAdministrator: true })
  expect(res.status).toBe(403)
  // and the privilege escalation did NOT happen
  const [row] = await testDb.select().from(user).where(eq(user.id, 'user-2'))
  expect(row.isAdministrator).toBe(0)
})

it('DELETE /api/users/:id returns 403 for an authenticated non-admin', async () => {
  const regular = insertUser('user-1', 'regular@example.com', { isAdministrator: false })
  const victim = insertUser('user-2', 'victim@example.com', { isAdministrator: false })
  mockSession(regular)
  const res = await request('DELETE', `/api/users/${victim.id}`)
  expect(res.status).toBe(403)
  const [row] = await testDb.select().from(user).where(eq(user.id, 'user-2'))
  expect(row).toBeDefined() // not deleted
})
```

- [ ] **Step 2: Run the tests — expect PASS** (the code already enforces this; these lock it)

Run: `npm run test:run -- src/server/routes/__tests__/users.test.ts`
Expected: PASS, including the 3 new cases. *If any FAIL, the authz is NOT actually in place — stop and escalate (that would be a live B8 vuln).*

- [ ] **Step 3: Commit**

```bash
git add src/server/routes/__tests__/users.test.ts
git commit -m "test(server): lock admin-only authz on user routes (403 for non-admins)" \
  -m "Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

### Task A2: Delete the leaked secret doc, PII file, and internal Office specs

**Files (delete, all git-tracked):**
- `CLAUDE-FIXES.md` (embeds a plaintext `BETTER_AUTH_SECRET` at line 44 + an exploit map)
- `httpsfreedium-mirror.cfdhttpsblog.devgenius.iothe-advanced-claude-code-setup-guide-358f7b69334d.txt` (maintainer PII)
- `Distributed Workflow JSON Specification.docx`
- `Trajectory Master Data Specification.docx`
- `Trajectory Project Plan.pptx`

- [ ] **Step 1: Remove the files**

```bash
cd C:/Trajectory/TrajectoryEditor
git rm "CLAUDE-FIXES.md" \
       "httpsfreedium-mirror.cfdhttpsblog.devgenius.iothe-advanced-claude-code-setup-guide-358f7b69334d.txt" \
       "Distributed Workflow JSON Specification.docx" \
       "Trajectory Master Data Specification.docx" \
       "Trajectory Project Plan.pptx"
```

- [ ] **Step 2: Verify none remain tracked**

Run: `git ls-files | grep -iE "CLAUDE-FIXES|freedium|\.docx$|\.pptx$"`
Expected: no output (exit 1).

- [ ] **Step 3: Commit**

```bash
git commit -m "chore: remove leaked secret doc, maintainer PII, and internal Office specs" \
  -m "CLAUDE-FIXES.md held a plaintext BETTER_AUTH_SECRET + an exploit map; the .txt held maintainer PII." # +trailer
```

> **USER ACTION (not a code change):** the deleted secret value `41f53b…` (64 hex) was committed historically. Repos are not yet public, so no history rewrite is needed — but if that exact value is the live secret in any real `.env`/deployment, **regenerate it** (`openssl rand -hex 32`) so the historical copy is worthless. The app reads the secret only from `process.env.BETTER_AUTH_SECRET` (`src/server/lib/auth.ts:12`), and `.env` is git-ignored.

### Task A3: Relicense to Apache-2.0 and remove the AGPL LICENSE.md

**Files:**
- Run script over the repo (rewrites ~529 source headers)
- Delete: `LICENSE.md` (AGPL/Saturnis)
- Modify: `package.json` (add `"license": "Apache-2.0"`)

- [ ] **Step 1: Apply the header rewrite**

Run: `node C:/Trajectory/_release-review-2026-06-04/tools/relicense-headers.mjs C:/Trajectory/TrajectoryEditor`
Expected: `... ; ~529 updated`.

- [ ] **Step 2: Remove the AGPL license file**

```bash
git rm LICENSE.md
```

- [ ] **Step 3: Add the license field to package.json**

In `C:/Trajectory/TrajectoryEditor/package.json`, add the license field immediately after the version line:
```json
  "version": "1.2.235",
  "license": "Apache-2.0",
```
(Apply the same insertion to `packages/ui/package.json`, `packages/tokens/package.json`, and `eslint-rules/package.json` — each currently has no `license` field; insert `"license": "Apache-2.0",` after their `"version": ...` line.)

- [ ] **Step 4: Verify no AGPL/Saturnis license text remains in shipped source**

Run: `grep -rIE "GNU AGPL|Saturnis\.io\. All rights reserved|See LICENSE\.md" src packages e2e electron --include=*.ts --include=*.tsx --include=*.mjs`
Expected: no output (exit 1). *(Branding image filenames containing "Saturnis" are brand assets, intentionally out of scope.)*

- [ ] **Step 5: Verify the app still builds**

Run: `npm run build`
Expected: build succeeds (header changes are comments only).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "license: standardize on Apache-2.0 (c) Dennis Brandl; drop AGPL LICENSE.md" # +trailer
```

### Task A4: Stop committing generated help.html + apply the ignore policy + untrack internal dirs

**Files:**
- Modify: `.gitignore` (append the §0.3 block + `public/help.html`)
- Untrack: `public/help.html`, `.planning/`, `.claude/`, `tasks/`, `docs/superpowers/`

- [ ] **Step 1: Append the ignore policy to `.gitignore`**

Append to `C:/Trajectory/TrajectoryEditor/.gitignore`:
```
# --- Public-release hygiene (internal-only; not shipped) ---
.planning/
.claude/
tasks/
docs/superpowers/
*.docx
*.pptx
*.continue-here.md

# Generated at build time (build:help); not tracked
public/help.html
```

- [ ] **Step 2: Untrack the now-ignored paths (keep them on disk)**

```bash
cd C:/Trajectory/TrajectoryEditor
git rm -r --cached --quiet public/help.html .planning .claude tasks docs/superpowers
```
Expected: lists the removed-from-index files. (Local files remain on disk; they're now ignored.)

- [ ] **Step 3: Verify the build regenerates help.html and the dirs are untracked**

Run: `npm run build:help && git status --short public/help.html`
Expected: `public/help.html` is regenerated and shows as ignored/untracked (no `M`/staged entry).
Run: `git ls-files | grep -E "^(\.planning/|\.claude/|tasks/|docs/superpowers/)" | head`
Expected: no output.

- [ ] **Step 4: Commit**

```bash
git add .gitignore
git commit -m "chore: stop tracking generated help.html and internal planning/.claude/docs dirs" # +trailer
```

### Task A5: Delete stray root files + remove the `_electron_native` stub + drop unused deps

**Files (delete, tracked):** `FixExtension.reg`, `NewMilestone.txt`, root nav PNGs (`Home.png`, `List.png`, `Next.png`, `Overview.png`, `Previous.png`), stray root schema JSONs (`managed_element.json`, `managed-workflow-step-base.json`, `master-action-library-schema.json`, `master-action-schema.json`, `master-environment-library-schema.json`, `master-environment-specification.json`, `master-workflow-library.json`, `master-workflow-step-library.json`), `_electron_native/package.json`.
**Modify:** `eslint.config.js` (drop the `_electron_native/**` ignore at line 38), `.dockerignore` (drop the `_electron_native` line), `package.json` (remove unused deps `react-textarea-autosize`, `radix-ui`, `concurrently`).

- [ ] **Step 1: Confirm the icon PNGs are NOT used by electron-builder before deleting**

Run: `grep -nE "Trajectory Icon|Home\.png|List\.png" electron-builder.yml electron/*.* 2>/dev/null`
Expected: no output → safe to delete. *(If any are referenced, KEEP those specific files and note it.)* Keep `Trajectory Icon 256.png`/`Trajectory Icon.png` only if referenced; otherwise delete with the rest.

- [ ] **Step 2: Delete the stray files**

```bash
cd C:/Trajectory/TrajectoryEditor
git rm "FixExtension.reg" "NewMilestone.txt" \
       "Home.png" "List.png" "Next.png" "Overview.png" "Previous.png" \
       "managed_element.json" "managed-workflow-step-base.json" \
       "master-action-library-schema.json" "master-action-schema.json" \
       "master-environment-library-schema.json" "master-environment-specification.json" \
       "master-workflow-library.json" "master-workflow-step-library.json" \
       "_electron_native/package.json"
```

- [ ] **Step 3: Remove the now-dangling references**

In `eslint.config.js`, delete the line containing `'_electron_native/**'` (≈line 38). In `.dockerignore`, delete the `_electron_native` line (≈line 6).

- [ ] **Step 4: Remove the three unused dependencies**

```bash
npm uninstall react-textarea-autosize radix-ui concurrently
```
Expected: package.json + package-lock.json updated. *(`concurrently` is in devDeps — confirm no npm script references it: `grep -n concurrently package.json` should show only its removal.)*

- [ ] **Step 5: Verify build + lint still pass**

Run: `npm run lint && npm run build`
Expected: both succeed.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "chore: remove stray root files, _electron_native stub, and unused deps" # +trailer
```

### Task A6: Add a root README

**Files:** Create `README.md`.

- [ ] **Step 1: Create `C:/Trajectory/TrajectoryEditor/README.md`**

```markdown
# Trajectory Editor

Visual editor for authoring distributed-workflow packages in the Trajectory suite. Built with React + Vite (and an optional Electron desktop shell), backed by a small Hono API server with SQLite (Drizzle) storage.

> **PLEASE NOTE:** Trajectory is a demonstration system, not intended for production environments. The editor and runtime are single-user systems and do not have the security necessary for production use. We recommend loading the applications into a Docker container for your testing.

## Quick start (Docker)

See [`DOCKER-README.md`](./DOCKER-README.md). In brief, from the suite umbrella directory:

```bash
docker compose up --build
```
Then open the Editor at the URL printed in `DOCKER-README.md`.

## Development

```bash
npm install
npm run dev      # client + API server
npm test         # vitest
```
Requires Node 22+.

## Documentation

- User guide: [`HELP.md`](./HELP.md)
- Docker: [`DOCKER-README.md`](./DOCKER-README.md)
- Workflow JSON spec: [`JSONSpec.md`](./JSONSpec.md)

## License

Apache-2.0 © 2026 Dennis Brandl. See [`LICENSE`](./LICENSE).
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add root README" # +trailer
```

### Task A7: Final verification + open PR (checkpoint)

- [ ] **Step 1: Full verify**

Run: `npm run lint && npm run test:run && npm run build`
Expected: all pass.

- [ ] **Step 2: Confirm nothing sensitive is tracked**

Run: `git ls-files | grep -iE "CLAUDE-FIXES|freedium|\.docx$|\.pptx$|settings\.local\.json|/\.planning/|/\.claude/"`
Expected: no output.

- [ ] **Step 3: Push + open PR — STOP for user review**

```bash
git push -u origin hardening/release
```
Then open a PR titled **"W1 publish-gate: Editor (secret/PII removal, Apache-2.0, hygiene, README)"**. **Do not merge** — hand to the user (sequential checkpoint).

---

## Phase B — TrajectoryRuntime PR  (`C:/Trajectory/TrajectoryRuntime`)

No root `package.json`; the web app is `engines/web-ui`, the reference engine is `engines/web`. Already on Node 22.

### Task B0: Branch

- [ ] **Step 1:**
```bash
cd C:/Trajectory/TrajectoryRuntime
git fetch origin
git switch -c hardening/release origin/main   # fallback: ... main
git status
```
Expected: `On branch hardening/release`.

### Task B1: Relicense to Apache-2.0 + remove AGPL LICENSE.md + flip package.json license fields

**Files:** run script (≈190 headers incl. `.kt`); delete `LICENSE.md`; modify `engines/web/package.json` and `engines/web-ui/package.json` (`"AGPL-3.0-or-later"` → `"Apache-2.0"`).

- [ ] **Step 1: Apply header rewrite**

Run: `node C:/Trajectory/_release-review-2026-06-04/tools/relicense-headers.mjs C:/Trajectory/TrajectoryRuntime`
Expected: `... ; ~190 updated` (covers `engines/web/src`, `engines/web-ui/src`, `engines/kmp-engine/src`, `engines/android-app/app/src`; generated `build/` trees are skipped).

- [ ] **Step 2: Remove AGPL license + flip the two license fields**

```bash
git rm LICENSE.md
```
In `engines/web/package.json` and `engines/web-ui/package.json`, change `"license": "AGPL-3.0-or-later"` → `"license": "Apache-2.0"`.

- [ ] **Step 3: Verify no AGPL/Saturnis license text remains in source**

Run: `grep -rIE "GNU AGPL|Saturnis\.io\. All rights reserved|See LICENSE\.md" engines/web/src engines/web-ui/src engines/kmp-engine/src engines/android-app/app/src`
Expected: no output. *(The `io.saturnis.trajectory` Android package namespace and "Saturnis Trajectory" logo files are brand, intentionally out of scope.)*

- [ ] **Step 4: Verify the web app builds**

Run: `cd engines/web-ui && npm run build`
Expected: build succeeds (comment-only header changes). *(KMP/Android Gradle build is verified in CI / W3, not here; the `.kt` changes are comments.)*

- [ ] **Step 5: Commit**

```bash
cd C:/Trajectory/TrajectoryRuntime
git add -A
git commit -m "license: standardize on Apache-2.0 (c) Dennis Brandl; drop AGPL LICENSE.md" # +trailer
```

### Task B2: Ignore policy + untrack internal dirs + stop tracking help.html

**Files:** modify `.gitignore`; untrack `engines/web-ui/public/help.html`, `.planning/`, `engines/web-ui/.planning/`, `.claude/`, `docs/superpowers/`.

- [ ] **Step 1: Append the ignore policy** to `C:/Trajectory/TrajectoryRuntime/.gitignore`:
```
# --- Public-release hygiene (internal-only; not shipped) ---
.planning/
**/.planning/
.claude/
docs/superpowers/
*.docx
*.pptx
*.continue-here.md

# Generated at build time (build:help); not tracked
engines/web-ui/public/help.html
```

- [ ] **Step 2: Untrack**
```bash
git rm -r --cached --quiet engines/web-ui/public/help.html .planning engines/web-ui/.planning .claude docs/superpowers
```
(If any path isn't present, omit it — run `git ls-files .planning engines/web-ui/.planning .claude docs/superpowers` first to see what exists.)

- [ ] **Step 3: Verify**

Run: `cd engines/web-ui && npm run build:help && cd ../.. && git ls-files | grep -E "\.planning/|/\.claude/|docs/superpowers/|web-ui/public/help\.html"`
Expected: build regenerates help.html; grep returns no output.

- [ ] **Step 4: Commit**
```bash
git add .gitignore
git commit -m "chore: stop tracking generated help.html and internal planning/.claude/docs dirs" # +trailer
```

### Task B3: Remove the abandoned iOS engines + the root spec docx

**Files (delete, tracked):** `engines/ios/` (56 files), `engines/ios-ui/` (16 files), `Trajectory Run Time Specification.docx`; modify `.dockerignore` (drop the now-redundant `engines/ios` and `engines/ios-ui` lines, ≈lines 10-11).

- [ ] **Step 1: Confirm no active build references** (belt-and-suspenders)

Run: `grep -rnE "engines/ios" --include=*.kts --include=Dockerfile --include=*.json . | grep -viE "\.dockerignore|docs/|\.planning/"`
Expected: only the `.dockerignore` lines (or nothing) — confirming no Gradle/workspace/Docker build depends on them.

- [ ] **Step 2: Delete**
```bash
git rm -r "engines/ios" "engines/ios-ui" "Trajectory Run Time Specification.docx"
```
Then remove the `engines/ios` and `engines/ios-ui` lines from `.dockerignore`.

- [ ] **Step 3: Verify the web build is unaffected**

Run: `cd engines/web-ui && npm run build && cd ../web && npm test`
Expected: both pass.

- [ ] **Step 4: Commit**
```bash
cd C:/Trajectory/TrajectoryRuntime
git add -A
git commit -m "chore: remove abandoned engines/ios + engines/ios-ui and internal spec docx" # +trailer
```

### Task B4: Add a root README + open PR (checkpoint)

- [ ] **Step 1: Create `C:/Trajectory/TrajectoryRuntime/README.md`**
```markdown
# Trajectory Runtime

Executes Trajectory distributed-workflow packages. Ships two parity-checked engines — `engines/web` (TypeScript reference + web product) and `engines/kmp-engine` (Kotlin Multiplatform, native) — plus a web runtime UI (`engines/web-ui`) and an Android app (`engines/android-app`).

> **PLEASE NOTE:** Trajectory is a demonstration system, not intended for production environments. The editor and runtime are single-user systems and do not have the security necessary for production use. We recommend loading the applications into a Docker container for your testing.

## Quick start (Docker)

See [`DOCKER-README.md`](./DOCKER-README.md). From the suite umbrella directory: `docker compose up --build`.

## Development

```bash
cd engines/web-ui && npm install && npm run dev   # runtime web UI
cd engines/web && npm install && npm test          # reference engine (node:test)
```
Requires Node 22+. The Kotlin engine builds via Gradle (`./gradlew.bat :jvmTest` in `engines/kmp-engine`).

## Documentation

- Workflow schema: [`SCHEMA-SPECIFICATION.md`](./SCHEMA-SPECIFICATION.md)
- Docker: [`DOCKER-README.md`](./DOCKER-README.md)

## License

Apache-2.0 © 2026 Dennis Brandl. See [`LICENSE`](./LICENSE).
```

- [ ] **Step 2: Commit, push, open PR — STOP for user**
```bash
git add README.md
git commit -m "docs: add root README" # +trailer
git push -u origin hardening/release
```
Open PR **"W1 publish-gate: Runtime (Apache-2.0, iOS removal, hygiene, README)"**. Do not merge.

---

## Phase C — TrajectoryActions PR  (`C:/Trajectory/TrajectoryActions`)

Action Container monorepo (`apps/console`, `packages/*`). **Has no `.dockerignore` — create one.** Node 20 → 22.

### Task C0: Branch
- [ ] **Step 1:**
```bash
cd C:/Trajectory/TrajectoryActions
git fetch origin
git switch -c hardening/release origin/main   # this repo is currently on main
git status
```
Expected: `On branch hardening/release`.

### Task C1: Relicense (single build-help.mjs header + dead LICENSE.md ref) + add license fields

**Files:** run script (fixes the one header in `apps/console/scripts/build-help.mjs` and its dead `See LICENSE.md`); modify `package.json` files to add `"license": "Apache-2.0"`.

- [ ] **Step 1: Apply header rewrite**

Run: `node C:/Trajectory/_release-review-2026-06-04/tools/relicense-headers.mjs C:/Trajectory/TrajectoryActions`
Expected: `... ; 1 updated` (the build-help.mjs header; the new line references `LICENSE`, fixing the dead `LICENSE.md` reference — this repo has no LICENSE.md).

- [ ] **Step 2: Add `"license": "Apache-2.0"`** after the `"version"` line in each of: `package.json`, `apps/console/package.json`, `packages/engine/package.json`, `packages/server/package.json`, `packages/storage/package.json`.

- [ ] **Step 3: Verify**

Run: `grep -rIE "GNU AGPL|Saturnis\.io|See LICENSE\.md" apps packages --include=*.ts --include=*.tsx --include=*.mjs`
Expected: no output.

- [ ] **Step 4: Commit**
```bash
git add -A
git commit -m "license: standardize on Apache-2.0 (c) Dennis Brandl; fix dead LICENSE.md ref" # +trailer
```

### Task C2: Create `.dockerignore` + ignore policy + untrack internal dirs + help.html

**Files:** create `.dockerignore`; modify `.gitignore`; untrack `apps/console/public/help.html`, `.planning/`, `.claude/`, `tasks/`, `docs/superpowers/`.

- [ ] **Step 1: Create `C:/Trajectory/TrajectoryActions/.dockerignore`**
```
node_modules
**/node_modules
dist
**/dist
.git
.claude
.planning
tasks
docs
*.docx
*.pptx
*.md
!apps/console/HELP.md
data
coverage
**/*.tsbuildinfo
```
*(Note: the umbrella compose builds the Action Container with the parent directory as context per its comment; this `.dockerignore` covers this repo when built standalone. Cross-context build context is a W5 concern.)*

- [ ] **Step 2: Append ignore policy** (§0.3) to `.gitignore`, plus:
```
# Generated at build time (build:help); not tracked
apps/console/public/help.html
```

- [ ] **Step 3: Untrack**
```bash
git rm -r --cached --quiet apps/console/public/help.html .planning .claude tasks docs/superpowers
```

- [ ] **Step 4: Verify**

Run: `cd apps/console && npm run build:help && cd ../.. && git ls-files | grep -E "\.planning/|/\.claude/|tasks/|docs/superpowers/|console/public/help\.html"`
Expected: regenerated; grep empty.

- [ ] **Step 5: Commit**
```bash
git add .dockerignore .gitignore
git commit -m "chore: add .dockerignore; stop tracking help.html and internal dirs" # +trailer
```

### Task C3: Delete junk + remove the orphaned `code-editor` page cluster + drop unused deps

**Files (delete, tracked):** `TrajectorySpec.md` (0 bytes), `ObservableStates.png`, `OpaqueStates.png`, `Action State Models.pptx`, and the orphan cluster `apps/console/src/features/code-editor/{CodeEditorPage,ObservableDiagram,OpaqueDiagram,StateButton}.tsx`.
**Modify:** `apps/console/src/App.tsx` (remove the `CodeEditorPage` import at ≈line 10 and the `<Route path="code-editor" .../>` at ≈line 33); `apps/console/package.json` (remove unused `@tanstack/react-table`, `vite-plugin-monaco-editor`).
**KEEP:** `InlineCodeEditorPage.tsx` (route `actions/:oid/code/:state`, reachable from `ActionDetailPage`/`TreeNode`/`SearchPanel`) and its siblings `hooks.ts`, `SaveDialog.tsx`, `TestPanel.tsx`, `VersionHistory.tsx`, `templates.ts`, and the `@monaco-editor/react` dep.

- [ ] **Step 1: Delete junk**
```bash
cd C:/Trajectory/TrajectoryActions
git rm "TrajectorySpec.md" "ObservableStates.png" "OpaqueStates.png" "Action State Models.pptx"
```

- [ ] **Step 2: Remove the orphaned page cluster**
```bash
git rm apps/console/src/features/code-editor/CodeEditorPage.tsx \
       apps/console/src/features/code-editor/ObservableDiagram.tsx \
       apps/console/src/features/code-editor/OpaqueDiagram.tsx \
       apps/console/src/features/code-editor/StateButton.tsx
```
Then in `apps/console/src/App.tsx`: delete the `import { CodeEditorPage } ...` line (≈10) and the `<Route path="code-editor" element={<CodeEditorPage />} />` line (≈33). Leave the `actions/:oid/code/:state` route (InlineCodeEditorPage) intact.

- [ ] **Step 3: Remove unused deps**
```bash
cd apps/console && npm uninstall @tanstack/react-table vite-plugin-monaco-editor && cd ../..
```

- [ ] **Step 4: Verify the console builds and tests pass** (catches any missed reference)

Run: `npm run build && npm test`
Expected: both succeed (no unresolved import of the deleted page/diagrams/deps).

- [ ] **Step 5: Commit**
```bash
git add -A
git commit -m "chore: remove empty spec, stray diagrams/pptx, orphaned code-editor page, unused deps" # +trailer
```

### Task C4: Bump Node 20 → 22 + add README + open PR (checkpoint)

**Files:** `Dockerfile` (lines 32, 67: `node:20-alpine` → `node:22-alpine`), `package.json` (line 39: `"node": ">=20.0.0"` → `">=22.0.0"`); create `README.md`.

- [ ] **Step 1: Bump Node** in `Dockerfile` (both `FROM node:20-alpine` lines → `node:22-alpine`) and `package.json` engines (`>=20.0.0` → `>=22.0.0`).

- [ ] **Step 2: Create `C:/Trajectory/TrajectoryActions/README.md`**
```markdown
# Trajectory Action Container

Hosts and executes Trajectory actions. A TypeScript monorepo — `apps/console` (management UI), `packages/server` (Trajectory REST protocol + management API), `packages/engine` (action state machine), `packages/storage` — with a Python sidecar that runs action code.

> **PLEASE NOTE:** Trajectory is a demonstration system, not intended for production environments. It does not have the security necessary for production use. We recommend running it in a Docker container, bound to localhost, for your testing.

## Quick start (Docker)

See [`DOCKER-README.md`](./DOCKER-README.md). From the suite umbrella directory: `docker compose up --build`.

## Development

```bash
npm install
npm run dev      # server + console
npm test         # vitest
```
Requires Node 22+.

## License

Apache-2.0 © 2026 Dennis Brandl. See [`LICENSE`](./LICENSE).
```

- [ ] **Step 3: Verify, commit, push, open PR — STOP for user**

Run: `npm run build`
Expected: succeeds.
```bash
git add -A
git commit -m "chore: bump Node 20->22; add root README" # +trailer
git push -u origin hardening/release
```
Open PR **"W1 publish-gate: Action Container (Apache-2.0, .dockerignore, dead-code + hygiene, README)"**. Do not merge.

---

## Phase D — TrajectoryActionTester PR  (`C:/Trajectory/TrajectoryActionTester`)

Smallest. Has a README already (needs fixes). Node 20 → 22.

### Task D0: Branch
- [ ] **Step 1:**
```bash
cd C:/Trajectory/TrajectoryActionTester
git fetch origin
git switch -c hardening/release origin/main   # fallback: ... main
git status
```

### Task D1: Relicense (single build-help.mjs header) + add license field
- [ ] **Step 1:** `node C:/Trajectory/_release-review-2026-06-04/tools/relicense-headers.mjs C:/Trajectory/TrajectoryActionTester`  → `1 updated`.
- [ ] **Step 2:** Add `"license": "Apache-2.0"` after the `"version": "0.1.0",` line in `package.json`.
- [ ] **Step 3 (verify):** `grep -rIE "GNU AGPL|Saturnis\.io|See LICENSE\.md" src scripts --include=*.ts --include=*.tsx --include=*.mjs` → no output.
- [ ] **Step 4 (commit):** `git add -A && git commit -m "license: standardize on Apache-2.0 (c) Dennis Brandl; fix dead LICENSE.md ref"` # +trailer

### Task D2: Ignore policy + untrack help.html + delete junk/redundant .gitkeep + drop msw
**Files:** modify `.gitignore`; untrack `public/help.html`; delete `test-output.txt` and the 5 redundant `.gitkeep` files; remove `msw` dep.

- [ ] **Step 1: Append to `.gitignore`:**
```
# --- Public-release hygiene ---
*.docx
*.pptx

# Generated at build time (build:help); not tracked
public/help.html
```
*(This repo has no `.planning/.claude/tasks/docs/superpowers` dirs.)*

- [ ] **Step 2: Untrack help.html + delete junk**
```bash
cd C:/Trajectory/TrajectoryActionTester
git rm --cached --quiet public/help.html
git rm "test-output.txt" \
       src/api/.gitkeep src/components/.gitkeep src/features/.gitkeep src/lib/.gitkeep src/store/.gitkeep
```

- [ ] **Step 3: Remove unused dep**
```bash
npm uninstall msw
```

- [ ] **Step 4: Verify**
Run: `npm run build:help && npm run build && npm test`
Expected: help regenerates; build + tests pass (msw was unused).

- [ ] **Step 5: Commit**
```bash
git add -A
git commit -m "chore: untrack generated help.html; remove test dump, redundant .gitkeep, unused msw" # +trailer
```

### Task D3: Fix the README (mislabel + Node + private path) + Node 20→22 + open PR (checkpoint)
**Files:** `README.md` (lines 7, 11, 44), `Dockerfile` (line 26: `node:20-alpine` → `node:22-alpine`).

- [ ] **Step 1: Edit `README.md`:**
  - Line 7: replace `Phase 4-01 scaffold — empty three-pane shell, single-file build pipeline.` with:
    `A working single-file tester for exercising any Trajectory Action Container's REST API.`
  - Line 11: replace `- Node 20 or later` with `- Node 22 or later`
  - Line 44 (the `## Spec` reference with the private `C:\TrajectoryActions\` path): replace the line with:
    `See the Action Container repository for the REST protocol specification.`
  - Add a disclaimer after the title:
    `> **PLEASE NOTE:** Trajectory is a demonstration system, not intended for production environments. This tester is a developer tool for exercising an Action Container's REST API.`
  - Add at the end:
    `## License`
    `Apache-2.0 © 2026 Dennis Brandl. See [LICENSE](./LICENSE).`

- [ ] **Step 2: Bump Node** in `Dockerfile` line 26 (`node:20-alpine` → `node:22-alpine`).

- [ ] **Step 3: Verify, commit, push, open PR — STOP for user**
Run: `npm run build`
Expected: succeeds.
```bash
git add -A
git commit -m "docs: fix README (description, Node 22, drop private path); bump Dockerfile Node" # +trailer
git push -u origin hardening/release
```
Open PR **"W1 publish-gate: Action Tester (Apache-2.0, README fix, hygiene, Node 22)"**. Do not merge.

---

## Self-review (performed against the spec)

**Spec coverage (W1 sections):**
- W1.1 secret/PII/exploit-map → Task A2 (delete) + A1 (lock authz via tests; grounding found authz already implemented) + USER-ACTION note on rotation. ✓
- W1.2 license (Apache-2.0, drop AGPL, headers, package.json, dead refs) → A3, B1, C1, D1. ✓ (Branding images/Android namespace explicitly out of scope — brand, not license.)
- W1.3 hygiene: stop committing help.html (A4/B2/C2/D2), ignore policy + untrack internal dirs (A4/B2/C2/D2), junk deletion (A5/C3/D2), dead-code removal (B3 ios, C3 code-editor), unused deps (A5/C3/D2), READMEs (A6/B4/C4/D3), Node 20→22 (C4/D3; Editor Dockerfile node:20.19 → also bump in A5? — see gap below). ✓ mostly
- Branching `hardening/release` off origin/main, per-repo PRs, checkpoints → A0/B0/C0/D0 + A7/B4/C4/D3. ✓

**Gap found + fixed inline:** the Editor Dockerfile uses `node:20.19-alpine` (lines 34, 66) but Phase A had no Node bump. **Add to Task A5 Step 3:** also change both `FROM node:20.19-alpine` lines in `C:/Trajectory/TrajectoryEditor/Dockerfile` to `node:22-alpine`, and re-verify with `npm run build`. (Runtime is already Node 22.)

**Placeholder scan:** no TBD/TODO; all deletions, edits, scripts, and commands are concrete. README bodies are complete.

**Type/identifier consistency:** test helpers (`insertUser`, `mockSession`, `request`, `testDb`, `user`, `eq`) match the verbatim harness from grounding; the relicense script's OLD/NEW strings match the verbatim headers; ignore-policy paths match each repo's verified tracked dirs.

**Note for executor:** repos are currently on feature branches (Editor `feat/catch-default-outputs`, Runtime `fix/web-ui-validator-…`, AT `feat/docker-setup`; AC on `main`). Branching off `origin/main` keeps W1 isolated from in-flight work; coordinate the eventual merge order with the user.
