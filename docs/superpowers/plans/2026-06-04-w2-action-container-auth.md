# W2 — Action Container Authentication & Hardening — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:executing-plans. TDD throughout. Steps use checkbox (`- [ ]`).

**Goal:** Close the unauthenticated-RCE chain in the Action Container: authenticate `/management/v1`, make the API key actually settable + default-on-when-configured, use a constant-time compare, allowlist CORS, cap decompression (zip-bomb), and default host exposure to loopback — without breaking the localhost console→server path.

**Architecture:** Auth model = **enforce when an `api_key` is configured (non-empty); open when not** — which is safe because host exposure defaults to **loopback at the compose layer** (the server keeps binding `0.0.0.0` inside the container so the console/nginx can still reach it). Operators enable auth by setting a key (UI or `ACTIONS_API_KEY` env); a boot-time warning fires if the API is reachable without one.

**Tech Stack:** Express + better-sqlite3 (`@trajectory/storage`), vitest + supertest, React console. Source: review report B1/B2/B3 + spec `2026-06-04-trajectory-demo-hardening-design.md`. Grounding captured 2026-06-04.

**Branch:** `hardening/ac-auth` off the current AC `hardening/release` (stacked). PR **base = `hardening/release`** so the diff is W2-only; merge the W1 PR (#3) first, then this.

**Commit trailer (every commit):** `-m "Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"`. AC has a husky pre-commit (eslint --fix + prettier on staged ts/tsx).

**Test command:** `npx vitest run --project server <path>` (and `--project storage`). Auth/settings tests should avoid creating instances (no Python subprocess needed).

---

## Design decisions (locked)

1. **`''`/`null` both mean "open"** in `auth.ts` (`if (!configuredKey)`), so a seeded empty key = open.
2. **Seed `api_key=''`** via a new `002` migration (`INSERT OR IGNORE`) so existing DBs gain the row; `001` also seeds it for fresh DBs.
3. **`ACTIONS_API_KEY` env**, if non-empty at boot, updates the `api_key` setting (lets exposed Docker enable auth declaratively).
4. **Constant-time compare** via `crypto.timingSafeEqual` with a length guard.
5. **CORS**: `ACTIONS_ALLOWED_ORIGINS` (comma list) if set; else default-allow **loopback origins only** (localhost/127.0.0.1/::1, any port) — keeps tester(3004)→container(3002) working while blocking drive-by from arbitrary sites.
6. **Redact** `api_key` in `GET /settings` (value → masked; expose a `configured: boolean`) and never echo it in the `PUT` response.
7. **Zip-bomb guard**: a shared `readZipEntryCapped()` + per-archive accounting (max total uncompressed bytes, max entry count, max single-entry bytes) wrapping all 6 `loadAsync` read sites.
8. **Loopback compose**: `127.0.0.1:3002:3002` / `127.0.0.1:3003:80` in AC + umbrella compose; add `ACTIONS_API_KEY` env passthrough. Server bind stays `0.0.0.0` (container-correct); add optional `HOST` env (default `0.0.0.0`) + the boot warning.

---

## Task T0 — Branch
- [ ] `cd C:/Trajectory/TrajectoryActions && git switch -c hardening/ac-auth hardening/release && git status` → on `hardening/ac-auth`, clean.

## Task T1 — `api_key` becomes a settable string setting (storage) [TDD]
**Files:** `packages/storage/src/repositories/settings.repository.ts`; `packages/storage/src/migrations/001-initial-schema.ts`; new `packages/storage/src/migrations/002-api-key-setting.ts` + register it; tests in `packages/storage/src/**/*.test.ts`.

- [ ] **Failing test** (storage): `settingsRepo.getValue('api_key')` returns `''` on a fresh DB; `settingsRepo.update('api_key','secret')` succeeds and `getValue` returns `'secret'`; `update('api_key','')` clears it. (Today it throws `NotFoundError`.)
- [ ] **Impl:**
  - Add `'api_key'` to `KNOWN_KEYS`.
  - In `validateValue`, add `case 'api_key': /* any string allowed; optional max length */ break` (before the throwing `default`).
  - In `001-initial-schema.ts` seed block, add `insertSetting.run('api_key', '', '', 'API key required on protected routes (empty = open)', 'string')`.
  - Add `002-api-key-setting.ts`: `db.prepare("INSERT OR IGNORE INTO settings (key,value,default_value,description,value_type) VALUES ('api_key','','','API key required on protected routes (empty = open)','string')").run()`. Register in the migrations list (follow `001`'s registration in the runner/index).
- [ ] **Verify:** `npx vitest run --project storage` green. **Commit:** `feat(storage): make api_key a settable string setting (+002 migration)`.

## Task T2 — Auth middleware: empty=open + constant-time compare [TDD]
**Files:** `packages/server/src/middleware/auth.ts`; new `packages/server/src/__tests__/auth.test.ts`.

- [ ] **Failing tests:** with `api_key` unset/`''` → `next()` called (open). With `api_key='secret'`: missing/`wrong` header → 401; correct header → pass. Timing-safe compare used (assert via behavior: differing-length keys → 401, no throw).
- [ ] **Impl:** replace the null check with `if (!configuredKey) { next(); return }`; replace `providedKey !== configuredKey` with a `timingSafeEqual` helper:
```ts
import { timingSafeEqual } from 'node:crypto'
function keysMatch(provided: unknown, expected: string): boolean {
  if (typeof provided !== 'string') return false
  const a = Buffer.from(provided), b = Buffer.from(expected)
  if (a.length !== b.length) return false
  return timingSafeEqual(a, b)
}
```
- [ ] **Verify:** `npx vitest run --project server packages/server/src/__tests__/auth.test.ts`. **Commit:** `fix(server): treat empty api_key as open; constant-time key compare`.

## Task T3 — Authenticate `/management/v1` [TDD] — the core fix
**Files:** `packages/server/src/index.ts`; `packages/server/src/__tests__/management-auth.test.ts` (new, with a light auth-aware test app — mirror `createTestApp` but add `app.use('/management/v1', createApiKeyAuth(settingsRepo))` before the router and seed `api_key`).

- [ ] **Failing test:** with `api_key='secret'` seeded, `GET /management/v1/dashboard` **without** `X-API-Key` → 401; **with** correct header → 200. `POST /management/v1/code/:oid/:state/test` (the RCE endpoint) → 401 without key.
- [ ] **Impl:** in `index.ts`, immediately before the `/management/v1` router mount (`:137`), add: `app.use('/management/v1', createApiKeyAuth(settingsRepo))` (mirrors `:124`). Keep `/health` above the auth line (it already is for `/trajectory`; ensure management health, if any, stays reachable — management has no separate health).
- [ ] **Verify** server tests green. **Commit:** `fix(server): require api_key on /management/v1 (closes unauth RCE/admin)`.

## Task T4 — `ACTIONS_API_KEY` env seeds the key at boot + exposure warning
**Files:** `packages/server/src/index.ts`.

- [ ] **Impl:** after repos init, before `app.listen`: if `process.env.ACTIONS_API_KEY` is a non-empty string, `settingsRepo.update('api_key', process.env.ACTIONS_API_KEY)`. Then compute `const host = process.env.HOST ?? '0.0.0.0'` and pass it to `app.listen(PORT, host, ...)`. After listen, if `settingsRepo.getValue('api_key')` is empty, `console.warn` a clear banner: API is UNAUTHENTICATED — bind to loopback or set ACTIONS_API_KEY before exposing.
- [ ] **Verify:** build + a small test that boot with `ACTIONS_API_KEY` set makes `getValue('api_key')` match (can unit-test the seeding helper if extracted). **Commit:** `feat(server): ACTIONS_API_KEY env + HOST bind + unauthenticated-exposure warning`.

## Task T5 — Redact `api_key` in settings API [TDD]
**Files:** `packages/server/src/routes/management.ts` (GET `/settings` ~:2664, PUT `/settings/:key` ~:2674).

- [ ] **Failing test:** `GET /management/v1/settings` returns the `api_key` item with its value **masked** (e.g. `value:''` + `configured:true` when set) — never the plaintext. `PUT /settings/api_key` response does **not** include the previous/!new plaintext.
- [ ] **Impl:** in GET, map settings so `api_key.value` → `''` and add `configured: <boolean>`; in PUT, when `key==='api_key'` omit `value`/`previous_value` from the response (return `{ key, applied:true }`).
- [ ] **Verify** server tests green. **Commit:** `fix(server): redact api_key in settings GET/PUT responses`.

## Task T6 — CORS allowlist [TDD]
**Files:** `packages/server/src/index.ts` (CORS block :93-100); test in `management-auth.test.ts` or new `cors.test.ts`.

- [ ] **Failing test:** request with `Origin: https://evil.example` → no `Access-Control-Allow-Origin: *`/echo (blocked); `Origin: http://localhost:3004` → allowed.
- [ ] **Impl:** replace `origin:'*'` with an origin function: allow when no Origin (same-origin/non-browser); if `ACTIONS_ALLOWED_ORIGINS` set, allow listed; else allow only loopback hostnames (`localhost`/`127.0.0.1`/`::1`). Keep `allowedHeaders` incl. `X-API-Key`.
- [ ] **Verify** green. **Commit:** `fix(server): replace wildcard CORS with loopback/allowlist origin policy`.

## Task T7 — Zip-bomb / decompression cap [TDD]
**Files:** new `packages/server/src/lib/safe-zip.ts`; apply in `packages/server/src/routes/export-import.ts` (:111, :511) and `packages/server/src/routes/management.ts` (:404, :484, :604, :912); test `packages/server/src/__tests__/safe-zip.test.ts`.

- [ ] **Failing test:** a crafted zip whose entries decompress beyond the cap (e.g. > 100 MB total or > N entries) is rejected with a `VALIDATION_ERROR` (code e.g. `ARCHIVE_TOO_LARGE`); a normal archive passes.
- [ ] **Impl:** `safe-zip.ts` exports `assertZipWithinLimits(zip, {maxEntries, maxTotalBytes, maxEntryBytes})` (sum `_data.uncompressedSize` across `zip.files`, enforce caps) and/or `readEntryCapped(entry, maxBytes)`. Call `assertZipWithinLimits` right after each `loadAsync` (defaults: maxEntries 5_000, maxTotalBytes 200MB, maxEntryBytes 50MB — tune). Extend the existing corruption catch blocks to surface the new error.
- [ ] **Verify** green. **Commit:** `fix(server): cap zip decompression size/count across all upload paths`.

## Task T8 — Console: API-key field in Settings (password input) + masked display
**Files:** `apps/console/src/features/settings/SettingsPage.tsx` (+ `SettingField` type branch); `apps/console/src/features/settings/hooks.ts`/`lib/api.ts` already round-trip any key.

- [ ] **Impl:** add `api_key` to `SETTING_LABELS` (label "API Key", description "Require this key (X-API-Key) on protected routes; empty = open"); render a `type="password"` input (add a `type` prop/branch to `SettingField`, default `"number"`); show a "configured / not set" hint from the GET `configured` flag; on save route through the existing `updateSetting` PUT. Do not pre-fill the real value (it's redacted).
- [ ] **Verify:** `cd apps/console && npm run build`. **Commit:** `feat(console): API key field in Settings (password, masked)`.

## Task T9 — Loopback compose + API-key env wiring
**Files:** `C:/Trajectory/TrajectoryActions/docker-compose.yml`; `C:/Trajectory/Trajectory/docker-compose.yml`; console nginx `apps/console/docker/default.conf.template` (optional key passthrough); `DOCKER-README.md` note.

- [ ] **Impl:** change published ports to `127.0.0.1:3002:3002` and `127.0.0.1:3003:80` in both compose files; add `environment: [ "ACTIONS_API_KEY=${ACTIONS_API_KEY:-}" ]` to the server service(s). Document in `DOCKER-README.md`: loopback by default; to expose, change the binding AND set `ACTIONS_API_KEY`. (Threading the key console→server via nginx `proxy_set_header X-API-Key ${ACTIONS_API_KEY}` is optional and only needed if the console must call an authed server — note as a follow-up; the console is same-origin behind nginx and the server reads the key from settings, so console calls still need the header — see open question.)
- [ ] **Verify:** compose `config` parses (`docker compose -f ... config` if docker available; else YAML lint). **Commit:** `chore(docker): bind AC services to loopback; ACTIONS_API_KEY passthrough`.

## Task T10 — Full verify + PR
- [ ] `npm run build` (root) + `npx vitest run --project server --project storage` green; `cd apps/console && npm run build` green.
- [ ] Push `hardening/ac-auth`; open PR **base `hardening/release`** titled "W2: Action Container authentication & hardening". STOP for user.

---

## Open questions / risks
- **Console→server auth when a key is set:** the console (browser, same-origin behind nginx) calls `/management/v1/*` with no key, so once a key is configured those calls would 401. Options: (a) the console reads/sends the key it just set (store in memory after save), or (b) nginx injects `X-API-Key` from `ACTIONS_API_KEY` env on proxied `/management/` + `/trajectory/` requests (server-side, key never in the browser). **(b) is preferred** — add it in T9. For loopback demos with no key, no impact. Flag for the PR; confirm with user if the console UX needs the key entered.
- Self-registration / multi-user is out of scope (single shared key — proportionate for a demo).
- `002` migration registration must match the repo's migration runner contract — verify the runner auto-discovers vs needs an explicit list entry.
- Self-review note: every task is independently testable; T3 depends on T1+T2; T8 depends on T5's `configured` flag.
