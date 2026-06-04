# Trajectory Demo — Full Hardening Design

**Date:** 2026-06-04
**Status:** Draft — awaiting user review
**Owner:** Dennis Brandl
**Source review:** `C:\Trajectory\_release-review-2026-06-04\RELEASE-READINESS-REPORT.md` (24-agent release review; blockers B1–B11)

## 1. Purpose & context

Harden the four-app Trajectory suite (Editor, Runtime, Action Container, Action Tester) for a **public demonstration release** distributed as **self-hostable** Docker images, where users **import third-party (untrusted) workflow packages** and the **Android app** is shipped. That deployment model reinstates a hostile threat model, so this initiative closes the security/legal/hygiene blockers from the release review while explicitly *not* building production-operations maturity (HA, audit logging, rate-limiting, multi-tenant isolation, support SLA), which the demonstration framing legitimately defers.

## 2. Locked decisions (from brainstorming)

1. **License:** standardize the entire suite on **Apache-2.0, © 2026 Dennis Brandl**. Permissive/fork-friendly for a self-hostable demo.
2. **Secret/PII removal:** repos are **not yet public**, so remove via a **plain delete commit** — *no* git-history rewrite, *no* force-push. Still regenerate the secret value, add guards against re-committing, and verify the code-level vulns the leaked playbook named are actually fixed.
3. **SCRIPT execution:** **off-by-default + per-import opt-in + warning** (web + Android). No Web Worker / Rhino-ClassShutter sandbox engineering.
4. **Execution model:** **sequential, per-repo `hardening/release` branch, one PR per workstream-per-repo, with a review checkpoint after each** (user reviews/merges).

## 3. Threat model & scope

**In scope (must close for the demo):** unauthenticated reachability of code-execution and destructive admin surfaces; rendering/execution of untrusted imported content (HTML, SCRIPT, action-server URLs, archives); leaked secret/PII in the published artifact; license/legal clarity; unsafe network defaults; showcase-grade repo hygiene.

**Out of scope (documented as known demo limitations):** TLS *termination* (ship reverse-proxy guidance + secure-cookie-under-HTTPS instead); production ops (HA, audit, rate-limit, backups); deep SCRIPT sandboxing (replaced by off-by-default); accessibility/polish beyond what blocks comprehension; performance.

**Standing assumption to re-validate:** "not yet public" is load-bearing for decision #2. If any repo was *ever* pushed to a remote others can read, the secret is compromised → history scrub + rotation reopen. **First task of W1 verifies this per repo before any delete.**

## 4. Workstreams

Each workstream = its own implementation plan + PR(s). Acceptance criteria are written test-first (TDD) where code is involved.

### W1 — Publish gate *(first; blocks any release; mostly fast)*

**W1.1 Secret / PII / exploit-map (Editor)**
- Pre-check: confirm no repo has a readable public remote (`git -C <repo> remote -v`; confirm with user). If any is public → STOP and escalate to history-scrub + rotation.
- Delete `TrajectoryEditor/httpsfreedium-mirror.cfd…358f7b69334d.txt` (maintainer PII) and `TrajectoryEditor/CLAUDE-FIXES.md` (exploit playbook + secret).
- **Regenerate** `BETTER_AUTH_SECRET`; ensure it is supplied only via env (`docker-compose.yml` already uses `${BETTER_AUTH_SECRET:?}` form — verify no plaintext default remains anywhere).
- **Code-half of B8 (must verify, independent of the doc):** confirm `TrajectoryEditor/src/server/routes/users.ts` `GET (~33)` / `PATCH (~53)` / `DELETE (~90)` enforce authorization (caller is admin or acting on self); add regression tests. Confirm no hardcoded secret remains in any `docker-compose.yml`.
- Add `.dockerignore` + `.gitignore` entries so these never re-enter the repo or image.

**W1.2 License standardization (all 4 repos)**
- Keep Apache-2.0 `LICENSE`; set appendix copyright to `Copyright 2026 Dennis Brandl`.
- Delete AGPL `LICENSE.md` in Editor + Runtime; remove dead `LICENSE.md` references (incl. `scripts/build-help.mjs` header in AC/AT).
- Normalize source-file headers (Editor ~591, Runtime ~192, AC/AT 1 each) from "AGPL v3 / © Saturnis.io / All rights reserved" → Apache-2.0 / © Dennis Brandl (scripted, reviewed).
- Add `"license": "Apache-2.0"` to every `package.json`.
- Reconcile splash/`HELP` images that render license/holder text.

**W1.3 Repo hygiene & READMEs (all 4 repos)**
- Stop committing generated help (`public/help.html`, `apps/console/public/help.html`): `.gitignore` them + `git rm --cached` + ensure CI/Docker builds them (`build:help`). Removes the perpetual dirty diffs.
- One shared `.gitignore`/`.dockerignore` policy dropping internal-only material from the public drop: `.planning/`, `.claude/` (incl. `settings.local.json`), `tasks/`, `docs/superpowers/`, `*.docx`, `*.pptx`, `*.continue-here.md`, mirrored `.TrajectoryActions/`/`.TrajectoryMobile/`.
- Delete junk: Editor `FixExtension.reg`, `NewMilestone.txt`, root duplicate nav PNGs + stray root JSON schemas, `_electron_native/` stub if unused; AC 0-byte `TrajectorySpec.md` + stray root `.png`/`.pptx`; AT `test-output.txt` + stray `.gitkeep`.
- Drop unused deps: AT `msw`; Editor `react-textarea-autosize`/`radix-ui`/`concurrently`; AC `@tanstack/react-table`/`vite-plugin-monaco-editor`.
- Remove abandoned trees from the public drop: Runtime `engines/ios` + `engines/ios-ui` (or clearly mark experimental); AC unreachable `code-editor` route (`apps/console/src/App.tsx:33`) and the components it drags in.
- Add a real root `README.md` (shared template) to Editor, Runtime, Action Container; fix the Action Tester README (remove "empty three-pane shell" line + the private `C:\…` path).
- Correct docs: "Node 20+" → "Node 22+" suite-wide.

*Acceptance:* `git ls-files` shows no secrets/PII/internal dirs; `docker build` image contains none of them (grep the built FS); each repo's existing test + lint suites green; every landing page renders a correct README.

### W2 — Action Container authentication *(the linchpin)*

- **B2 — make auth enableable (precondition for everything else):**
  - Seed/allow `api_key` in `packages/storage/src/migrations/001-initial-schema.ts` and add it to `KNOWN_KEYS` in `packages/storage/src/repositories/settings.repository.ts:5-10` so `update()` no longer throws for it.
  - Add a Settings UI field in `apps/console/.../settings/SettingsPage.tsx`.
  - Constant-time compare in `packages/server/src/middleware/auth.ts` (replace `!==` at `:28`).
  - **Default-enabled** whenever the server is bound to a non-loopback interface.
- **B1 — protect management:** apply the auth middleware to the entire `/management/v1` router (`packages/server/src/index.ts:137-151`), matching `/trajectory/v1` (`:124`). The `code/:oid/:state/test` exec endpoint (`management.ts` → `instance-manager.ts testCode` → `python-sidecar/sandbox_runner.py` `exec`) is then unreachable unauthenticated.
- **B3 — CORS + destructive admin:** replace wildcard `origin:'*'` (`index.ts:93-100`) with an env-configured allowlist; the destructive `snapshot/import` and exfiltrating `snapshot/export` (`routes/export-import.ts:188-277,493-647`) are then both behind auth + same-origin.
- **Zip-bomb guard (critic-added):** add a decompression-ratio/size cap to AC ZIP paths (`export-import.ts:511`, upload branches), mirroring the Runtime's cap.
- **Default binding:** loopback in the AC `docker-compose.yml` + the umbrella compose (see W5).

*Acceptance (test-first):* unauth request to any `/management/*` → 401; `code/test` rejects without a valid key; key settable + togglable via settings; CORS rejects non-allowlisted origins; oversized/overcompressed archive rejected.

### W3 — Runtime untrusted-content hardening

- **B5 — stored XSS:** introduce a shared `<SafeHtml>` sanitizer (DOMPurify) reused by `engines/web-ui/src/components/elements/TextElement.tsx` and `HeaderElement.tsx` (currently raw `dangerouslySetInnerHTML` at `:61`/`:63`); ensure `utils/richText.ts:36-39` escapes substituted `{{mustache}}` values; add `dompurify` dependency. Mirror the Editor's existing `sanitizeHTML()` (`TextRenderer.tsx:199` / `HeaderRenderer.tsx:187`). Ship a CSP in the Runtime nginx image (`Dockerfile`). Add a lint rule banning bare `dangerouslySetInnerHTML`.
- **B4 — SCRIPT off-by-default:** gate execution in `engines/web/src/step-handlers.ts:170` (`new Function`) and the KMP `ScriptExecutor` (`.js.kt`, `.jvm.kt`) behind an explicit per-import "allow SCRIPT execution" flag (default off) + a clear warning at import/enable time; when disabled, SCRIPT steps fail closed with a clear status (reuse the per-step fail signal).
- **B6 — SSRF:** add an action-server allowlist + scheme check + private-IP/loopback block in **both** validators (`engines/web/src/validator.ts`, `engines/web-ui/src/manager/validation.ts`) and at the client (`actionProxy/ActionApiClient.ts:65-103`); stop auto-firing `fetchAllCapabilities` on workflow open (`WorkflowCoordinator.ts:282,290` / `useServerBindings.ts:43-44`) — require explicit user intent / confirmation for non-loopback hosts.

*Acceptance (test-first):* a Text/Header element containing `<img src=x onerror=…>` renders inert; SCRIPT step does not execute unless enabled, and shows a clear disabled state; proxy/validators reject non-allowlisted/private/non-http(s) URIs; no network call fires on import without intent.

### W4 — Android hardening *(Runtime/engines/android-app)*

- **B7 — Zip Slip:** in `app/.../util/FileProcessor.kt:63-106`, canonicalize each entry and assert it resolves under `outputDir` before writing (`:68/:70`); reject `..`/absolute entries. Unit test with a malicious entry name.
- **Cleartext:** add a scoped `networkSecurityConfig` (allow cleartext only to explicitly configured/loopback hosts) rather than a blanket `usesCleartextTraffic=true`; keeps B6 from being aggravated.
- SCRIPT off-by-default (W3) applies to the Android Rhino path.

*Acceptance:* archive entry `../evil` is rejected; manifest has a scoped network-security config; SCRIPT disabled by default on device.

### W5 — Deploy / transport posture

- **Loopback-default bindings:** change published ports to `127.0.0.1:PORT:PORT` in the umbrella `Trajectory/docker-compose.yml` (currently `0.0.0.0` via short form) and per-app composes; document an explicit opt-in to expose (which turns on AC auth + advises TLS).
- **TLS guidance (not build):** document running behind a TLS-terminating reverse proxy; make better-auth secure cookies engage when `baseURL` is HTTPS (`TrajectoryEditor/src/server/lib/auth.ts:10`).
- **Self-registration:** default closed/invite when bound to a non-loopback interface; open sign-up remains fine on loopback (`auth.ts:34-47`).
- **Quickstart fix:** the umbrella compose defaults `BETTER_AUTH_SECRET` to empty → Editor crash-loops; generate-on-first-run or document a required `.env` clearly.
- **Disclaimer:** elevate the existing "demonstration system, not for production" `HELP.md` disclaimer into every README + a runtime startup log banner.

*Acceptance:* fresh `docker compose up` binds only to loopback and starts cleanly with documented `.env`; exposing requires a deliberate change that prompts enabling auth; READMEs + startup logs carry the disclaimer.

## 5. Cross-cutting

- **Branching:** per repo, a `hardening/release` branch off `origin/main`, isolated from in-flight feature branches (Editor `feat/catch-default-outputs`, Runtime `fix/web-ui-validator-…`, AT `feat/docker-setup`); coordinate/rebase with the user at merge time. One PR per workstream-per-repo.
- **TDD + verification:** failing test → fix → green; atomic commits; each PR gates on the repo's existing tests + lint + a clean `docker build`. Per the verification-before-completion discipline, claims of "fixed" require shown passing output.
- **CI caveat:** TrajectoryActions CI is known-broken on a cross-repo `@trajectory/ui` dep (deferred 2026-05-21) — do not block W2 on CI; verify locally and note status.
- **Duplicated validators/specs:** B6 touches both Runtime validators (web + web-ui) which are known to drift; keep them in lockstep. (Longer-term shared-parser/`<SafeHtml>` consolidation is recommended but only the shared `<SafeHtml>` is in scope here.)
- **No merges/pushes to `main` without the user.**

## 6. Boundaries (user responsibilities)

- Confirm "not yet public" per repo (W1.1 pre-check); decide history-scrub + rotation if that turns out false.
- Review and merge each PR; run any real deployment.
- Final license/legal sign-off on the Apache-2.0 normalization.

## 7. Sequencing & checkpoints

W1 (publish gate) → **checkpoint** → W2 (AC auth) → **checkpoint** → W3 + W4 (Runtime + Android untrusted-content) → **checkpoint** → W5 (deploy posture) → **final re-review**.

## 8. Success criteria

- B1–B11 closed *for the demo threat model*, verified: AC management API authenticated + loopback-default + CORS-allowlisted; Runtime sanitizes untrusted HTML (+ CSP), SCRIPT off-by-default, SSRF-guarded; Android Zip-Slip closed + scoped cleartext; Editor secret/PII gone + `users.ts` authz verified + secure-cookie-under-HTTPS; suite on a single coherent Apache-2.0 license; clean, professional, README'd public repos with the demo disclaimer prominent.
- **Re-review gate:** re-run the Security + Documentation critics against the Action Container and Runtime after W1–W4; suite reaches **GO (demo)** when all of the above verify.

## 9. Open questions / risks

- Header-normalization scope (~599 files) is scripted — risk of touching generated/vendored files; the script must respect each repo's `ignoreDirs` from the review maps.
- Removing `engines/ios*` and the AC `code-editor` route assumes they are truly unwired (review found them orphaned) — confirm no hidden references before deletion.
- SCRIPT "off-by-default" must not break the curated sample/demo workflows that legitimately use SCRIPT — provide a clear enable path and update samples/docs.
