# W3 — Runtime Untrusted-Content Hardening — Implementation Plan

> REQUIRED SUB-SKILL: superpowers:executing-plans. TDD. Checkbox steps.

**Goal:** Make the Runtime safe to open untrusted `.WFmasterX` packages: sanitize author HTML + ship a CSP (B5), execute SCRIPT steps only on explicit opt-in (B4), and guard the action-proxy against SSRF (B6).

**Repo/branch:** `TrajectoryRuntime`, new branch `hardening/runtime-untrusted` off `hardening/release` (W1, stacked). PR **base = `hardening/release`**.

**Test runners (node:test, NOT vitest):** web engine `cd engines/web && npm test`; web-ui `cd engines/web-ui && npm test` (and `node --import tsx --test <file>` for one file). No DOM infra in web-ui → unit-test pure functions; mirror the Editor's *tested* sanitizer for DOMPurify.

**Commit trailer:** every commit ends with `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`. (Runtime has no husky.)

---

## T0 — Branch
- [ ] `cd C:/Trajectory/TrajectoryRuntime && git switch -c hardening/runtime-untrusted hardening/release`

## T1 — B5: sanitize author HTML + escape substituted values
**Files:** add `engines/web-ui/package.json` deps `dompurify ^3.3.3` + `@types/dompurify ^3.0.5`; new `engines/web-ui/src/utils/sanitize-html.ts` (mirror `TrajectoryEditor/src/lib/sanitize-html.ts` — same ALLOWED_TAGS/ATTR + `data-param-chip`); modify `engines/web-ui/src/utils/richText.ts` (escape the `{{mustache}}` branch); `engines/web-ui/src/components/elements/TextElement.tsx` + `HeaderElement.tsx` (wrap `substituteChips(...)` in `sanitizeHTML(...)`). Test `engines/web-ui/src/utils/richText.test.ts`.

- [ ] **Failing test** (richText): a property value `<img src=x onerror=alert(1)>` substituted via `{{Key}}` is HTML-escaped in the output (no raw `onerror`). (Today the mustache branch returns it raw.)
- [ ] **Impl:** in `richText.ts` mustache branch, wrap the value: `const v = lookup(trimmed); return v !== undefined ? escapeHtml(v) : \`{{${trimmed}}}\``. Create `sanitize-html.ts` (DOMPurify, mirroring the Editor). In `TextElement.tsx`/`HeaderElement.tsx` change `const html = substituteChips(...)` → `const html = sanitizeHTML(substituteChips(...))`; import `sanitizeHTML`. `npm install` to add deps.
- [ ] **Verify:** `cd engines/web-ui && node --import tsx --test src/utils/richText.test.ts` green; `npm run build` green. **Commit:** `fix(web-ui): sanitize author HTML (DOMPurify) + escape substituted values`.

## T2 — B5: ship a CSP in the Runtime nginx image
**Files:** `Dockerfile` (the inline `printf` nginx server block, ~:70-78).
- [ ] **Impl:** add `add_header Content-Security-Policy "default-src '\''self'\''; img-src '\''self'\'' data:; style-src '\''self'\'' '\''unsafe-inline'\''; script-src '\''self'\''; object-src '\''none'\''; base-uri '\''self'\''; frame-ancestors '\''none'\''" always;` inside the server block (after `index index.html;`). `style-src unsafe-inline` is required (inline `style={{}}` + `<style>` tags). Keep the `printf`/escaping consistent (single-quoted format string → escape inner single quotes).
- [ ] **Verify:** Dockerfile parses (visual; no Docker on box). **Commit:** `fix(runtime): ship a Content-Security-Policy header in the nginx image`.

## T3 — B4: SCRIPT execution off-by-default + opt-in
**Files:** `engines/web/src/engine.ts` (ctor `setup` type + private field + gate at the SCRIPT dispatch ~:1074); `engines/web-ui/src/coordinator/WorkflowCoordinator.ts` (thread `allowScriptExecution` into `engineSetup`, ~:106-124, from a persisted setting); a tiny `engines/web-ui/src/settings.ts` (localStorage-backed `getAllowScript()/setAllowScript()`, default false) + a UI toggle on the Home/settings surface with a warning. Tests in `engines/web/src/engine-script.test.ts`.

- [ ] **Failing test** (engine): a workflow with a SCRIPT step whose `script_config.source` sets an output, run with default setup (no `allowScriptExecution`), ends `ERRORED` (or the SCRIPT step does not execute / output not written). With `new WorkflowEngine(wf, { allowScriptExecution: true })` it executes as today.
- [ ] **Impl:** add `allowScriptExecution?: boolean` to the ctor `setup` type; store `this.allowScript = setup?.allowScriptExecution ?? false`. At `engine.ts:1074` SCRIPT block: if `target.step.script_config?.source` and `!this.allowScript` → record trace `ERRORED` with message `SCRIPT execution is disabled` and set states ERRORED (mirror the existing failure path), return. Thread the flag from `WorkflowCoordinator.start()` engineSetup (read `getAllowScript()`). Add the localStorage setting + a labelled toggle ("Allow SCRIPT execution from imported workflows — only enable for trusted packages") with a warning.
- [ ] **Verify:** `cd engines/web && npm test` green (incl. new gate tests + existing SCRIPT tests pass with the flag); `cd engines/web-ui && npm run build`. **Commit:** `feat(runtime): SCRIPT execution off by default; per-session opt-in`.

## T4 — B6: SSRF guard on action-server URIs
**Files:** new `engines/web/src/lib/server-uri.ts` (`assertAllowedServerUri(uri, allowlist?)` → throws/returns reason; scheme http(s) only; default allow loopback (localhost/127.0.0.1/::1) + an optional allowlist; reject otherwise incl. private/metadata ranges) — exported from `engines/web`; mirror/import in `engines/web-ui`. Apply in `engines/web/src/validator.ts` (new phase after `actionProxyValidation`, ~:668) AND `engines/web-ui/src/manager/validation.ts` (before `structuralValidation`, ~:442); apply at the fetch chokepoints: `engines/web-ui/src/actionProxy/ActionApiClient.ts` (guard each of the 5 methods) and the 2 raw fetches in `WorkflowCoordinator.ts` (:282/:290); guard `useServerBindings.ts` before `getCapabilities` (don't auto-fire to a disallowed URI). Tests: `engines/web/src/lib/server-uri.test.ts` + extend `ActionApiClient.test.ts`.

- [ ] **Failing tests:** `assertAllowedServerUri('http://localhost:3002')` ok; `'https://evil.example'` rejected; `'file:///etc/passwd'` rejected; `'http://169.254.169.254/'` rejected; allowlist honored. `ActionApiClient.invoke('https://evil.example', ...)` throws before any fetch.
- [ ] **Impl:** write `server-uri.ts`; call it in both validators (return a `ValidationResult` error on a bad URI — iterate `environment_specifications[].action_server_specifications[].uri`); add a guard at the top of each `ActionApiClient` method (and the 2 raw fetches + before `getCapabilities`). Allowlist source: a persisted setting/env (default empty → loopback-only).
- [ ] **Verify:** `cd engines/web && npm test`; `cd engines/web-ui && npm test`; both builds. Existing ActionApiClient tests use `http://localhost:3002` (loopback) → stay green. **Commit:** `fix(runtime): SSRF guard — validate action-server URIs (scheme + loopback/allowlist)`.

## T5 — Full verify + PR
- [ ] `cd engines/web && npm test` + `cd engines/web-ui && npm test` + both `npm run build` green.
- [ ] Push `hardening/runtime-untrusted`; open PR base `hardening/release` titled "W3: Runtime untrusted-content hardening (XSS, SCRIPT, SSRF)". STOP for user.

---

## Notes / risks
- Both Runtime validators are duplicated/drifted ([[project_runtime_duplicate_validators]]); the URI guard must go in BOTH.
- KMP/Android SCRIPT (Rhino) + the Ktor SSRF + Android Zip-Slip are **W4** (this is web-only).
- DOMPurify needs a DOM → runs in the browser (real runtime); not unit-tested under node:test (mirrors the Editor's tested config). The substituted-value escape IS unit-tested.
- CSP `style-src 'unsafe-inline'` is required by the inline-style rendering; documented trade-off.
