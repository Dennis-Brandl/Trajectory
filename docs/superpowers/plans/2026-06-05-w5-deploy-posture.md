# W5 — Deploy / Transport Posture — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:executing-plans. TDD where code is involved. Checkbox steps.

**Goal:** Make the demo safe to self-host: bind services to loopback by default, gate open self-registration + secure cookies on the Editor's exposure, and surface the "demonstration system" disclaimer — without breaking the zero-config local `docker compose up`.

**Locked decisions (from this session):** (1) **auto-generate the auth secret on first run** — ALREADY implemented in `TrajectoryEditor/docker-entrypoint.sh:19-31` (generates + persists `/data/.auth-secret`), so the umbrella compose's empty `BETTER_AUTH_SECRET` does NOT crash-loop; W5 only documents this. (2) **self-signup closed when exposed, open on loopback** — derive from `BASE_URL`, override via `ALLOW_OPEN_SIGNUP=true`.

**Repos/branches:**
- **Editor** — new branch `hardening/deploy` off `hardening/release` (W1, stacked). PR **base = `hardening/release`**.
- **Umbrella `Trajectory`** — commit on the existing local `docs/hardening-spec` branch (not pushed; the user's super-repo). Leave the pre-existing dirty files (`RESUME.md`, try-catch plans) alone.

**Test/build:** Editor uses **vitest** (`npm test`) + `tsc -b` (in `npm run build`). No Docker on this box → compose changes are inspection-verified; flag `docker compose up` for the user.

**Commit trailer:** every commit ends with `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`. (Editor has a husky pre-commit that bumps patch version + runs prettier — expect an auto-amended version bump.)

---

## SCOPE

**Core (this pass — verifiable here):** Editor auth gating (secure cookies + self-signup) + startup banner; Editor standalone compose loopback; umbrella compose loopback + quickstart doc; Editor README disclaimer + TLS guidance.

**Deferred (flag for user — mechanical sweep / not verifiable here):**
- Per-app standalone composes loopback for **Runtime / Actions / AT** + their README disclaimers (each on a different `hardening/*` branch → separate commits/PRs).
- `docker compose up --build` runtime validation (no Docker on this box).
- **RELEASE GATE: PII/secret git-history scrub (or fresh snapshot) before any repo is made public** — user-owned per spec §6.

---

## T0 — Editor branch
- [ ] `cd C:/Trajectory/TrajectoryEditor && git switch -c hardening/deploy hardening/release`

## T1 — Deploy-policy helpers (pure, testable)
**Files:** Create `src/server/lib/deploy-policy.ts`; Test `src/server/lib/deploy-policy.test.ts`.

- [ ] **Failing test** (`deploy-policy.test.ts`):
```ts
import { describe, it, expect } from 'vitest'
import { shouldUseSecureCookies, isOpenSignupAllowed } from './deploy-policy'

describe('shouldUseSecureCookies', () => {
  it('true for https base URLs', () => expect(shouldUseSecureCookies('https://demo.example')).toBe(true))
  it('false for http base URLs', () => expect(shouldUseSecureCookies('http://localhost:3000')).toBe(false))
})

describe('isOpenSignupAllowed', () => {
  it('open on loopback hosts', () => {
    expect(isOpenSignupAllowed('http://localhost:3000', undefined)).toBe(true)
    expect(isOpenSignupAllowed('http://127.0.0.1:3000', undefined)).toBe(true)
  })
  it('closed when exposed on a non-loopback host', () => {
    expect(isOpenSignupAllowed('https://demo.example', undefined)).toBe(false)
    expect(isOpenSignupAllowed('http://192.168.1.50:3000', undefined)).toBe(false)
  })
  it('explicit ALLOW_OPEN_SIGNUP=true overrides', () => {
    expect(isOpenSignupAllowed('https://demo.example', 'true')).toBe(true)
  })
})
```
- [ ] **Run:** `cd C:/Trajectory/TrajectoryEditor && npm test -- deploy-policy` → FAIL (module missing).
- [ ] **Impl** (`deploy-policy.ts`):
```ts
// Copyright (c) 2026 Dennis Brandl
// Licensed under the Apache License, Version 2.0. See LICENSE for details.

/** Secure cookies require HTTPS; engage them only when the public base URL is https. */
export function shouldUseSecureCookies(baseUrl: string): boolean {
  return baseUrl.startsWith('https://')
}

/**
 * Open self-registration is allowed on loopback (easy local demo) or when an
 * operator explicitly opts in (ALLOW_OPEN_SIGNUP=true). It is CLOSED by default
 * once the Editor is exposed on a non-loopback host, so a public demo does not
 * accept anonymous account creation (the first account becomes admin).
 */
export function isOpenSignupAllowed(baseUrl: string, allowFlag: string | undefined): boolean {
  if (allowFlag === 'true') return true
  try {
    const host = new URL(baseUrl).hostname.replace(/^\[|\]$/g, '')
    return host === 'localhost' || host === '127.0.0.1' || host === '::1'
  } catch {
    return false
  }
}
```
- [ ] **Run:** `npm test -- deploy-policy` → PASS. **Commit:** `feat(editor): deploy-policy helpers (secure cookies + signup gating)`.

## T2 — Wire policy into better-auth
**Files:** Modify `src/server/lib/auth.ts`.

- [ ] **Impl:** import the helpers and apply them:
```ts
import { shouldUseSecureCookies, isOpenSignupAllowed } from './deploy-policy'
```
Compute after `baseURL` is defined:
```ts
const openSignup = isOpenSignupAllowed(baseURL, process.env.ALLOW_OPEN_SIGNUP)
const useSecureCookies = shouldUseSecureCookies(baseURL)
```
Change `emailAndPassword`:
```ts
  emailAndPassword: {
    enabled: true,
    disableSignUp: !openSignup,
  },
```
Add an `advanced` block (place after `emailAndPassword`):
```ts
  advanced: {
    useSecureCookies,
  },
```
- [ ] **Verify:** `cd C:/Trajectory/TrajectoryEditor && npm run build` (tsc -b validates the better-auth option names) → green. **Commit:** `feat(editor): gate self-signup + secure cookies on exposure (auth.ts)`.

## T3 — Startup posture banner
**Files:** Modify `src/server/index.ts` (the `serve(...)` callback, ~:145-150).

- [ ] **Impl:** inside the listen callback, after `console.log(\`Server running on port ${port}\`)`, add:
```ts
    const baseURL = process.env.BASE_URL || 'http://localhost:3000'
    const { shouldUseSecureCookies, isOpenSignupAllowed } = await import('./lib/deploy-policy')
    const openSignup = isOpenSignupAllowed(baseURL, process.env.ALLOW_OPEN_SIGNUP)
    console.log('────────────────────────────────────────────')
    console.log('Trajectory Editor — DEMONSTRATION system, not hardened for production.')
    console.log(`  Sign-up: ${openSignup ? 'OPEN' : 'CLOSED (set ALLOW_OPEN_SIGNUP=true to allow)'}`)
    console.log(`  Secure cookies: ${shouldUseSecureCookies(baseURL) ? 'on (HTTPS)' : 'off (HTTP — put behind a TLS proxy to expose)'}`)
    console.log('────────────────────────────────────────────')
```
(Use a static `import` at the top instead of `await import` if the callback is not async — check the callback signature; the helpers are side-effect-free either way.)
- [ ] **Verify:** `npm run build` green. **Commit:** `feat(editor): demonstration-system + security-posture startup banner`.

## T4 — Editor standalone compose → loopback
**Files:** Modify `TrajectoryEditor/docker-compose.yml`.
- [ ] **Impl:** change the published port from `"3000:3000"` to `"127.0.0.1:3000:3000"`. Add a comment: `# Loopback-only by default. To expose, change to "0.0.0.0:3000:3000" AND set BASE_URL=https://…, ALLOWED_ORIGINS, and run behind a TLS proxy.`
- [ ] **Verify:** YAML inspection. **Commit:** `fix(editor): bind compose port to loopback by default`.

## T5 — Editor README disclaimer + TLS guidance
**Files:** Modify `TrajectoryEditor/README.md`.
- [ ] **Impl:** ensure a prominent top-of-file note: "⚠️ Demonstration system — not hardened for production." Add a short "Exposing beyond localhost" section: set `BASE_URL=https://…`, `ALLOWED_ORIGINS`, run behind a TLS-terminating reverse proxy (secure cookies engage automatically under HTTPS); sign-up auto-closes when exposed (`ALLOW_OPEN_SIGNUP=true` to override); the auth secret auto-generates into the `/data` volume on first run (set `BETTER_AUTH_SECRET` to pin it).
- [ ] **Verify:** read-back. **Commit:** `docs(editor): demonstration disclaimer + exposing/TLS guidance`.

## T6 — Editor full verify + PR
- [ ] `cd C:/Trajectory/TrajectoryEditor && npm run build && npm test` green (incl. new deploy-policy tests; existing suite unbroken).
- [ ] Push `hardening/deploy`; open PR base `hardening/release` titled "W5: Editor deploy posture (loopback, signup/secure-cookie gating, disclaimer)". STOP for user.

## T7 — Umbrella compose → loopback + quickstart doc
**Files:** Modify `C:/Trajectory/Trajectory/docker-compose.yml`.
- [ ] **Impl:** prefix every published port with `127.0.0.1:` (editor 3000, runtime 3001, actions-server 3002, actions-console 3003, tester 3004). Update the header `Access:` comment to note loopback-only + how to expose (per-service). Add a note that `BETTER_AUTH_SECRET` auto-generates on first run (leave the `${BETTER_AUTH_SECRET:-}` default) and that exposing requires enabling AC auth (`ACTIONS_API_KEY`) + TLS.
- [ ] **Verify:** YAML inspection (no Docker here → flag `docker compose up --build` for the user).
- [ ] **Commit** on `docs/hardening-spec` (umbrella): `fix(compose): bind all services to loopback by default + quickstart notes`. Also commit the W4 + W5 plan docs that are dangling. Leave pre-existing dirty files.

---

## Notes / risks
- better-auth auto-enables secure cookies under https already; `advanced.useSecureCookies` makes it explicit + drives the banner. If `tsc` rejects either option name on 1.4.18, check `dist/**/*.d.mts` for the exact key and adjust (build is the gate).
- Editor husky pre-commit bumps the patch version + prettier on every commit — that's expected, not a failure.
- Deferred per-app composes/READMEs span Runtime (`hardening/android`), Actions (`hardening/ac-auth`), AT (`hardening/release`) — do as small per-branch commits when the user confirms.

## Self-review
- Spec W5 coverage: loopback bindings → T4 (Editor) + T7 (umbrella), per-app deferred; TLS guidance + secure-cookie-under-HTTPS → T2 + T5; self-registration closed-when-exposed → T2; quickstart secret → already done (entrypoint), documented in T5/T7; disclaimer in README + startup banner → T3 + T5. History-scrub gate = deferred/user-owned.
- No placeholders: helper code, auth wiring, banner, and compose edits are all concrete. The only env-specific check is the better-auth option names, gated by `tsc`.
