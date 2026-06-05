# W4 — Android / KMP Hardening — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:executing-plans (inline) or superpowers:subagent-driven-development. TDD. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Make the Android runtime safe to open untrusted `.WFmasterX` packages: reject Zip-Slip archive entries (B7), execute SCRIPT only on explicit opt-in in the KMP engine (B4 parity with W3-web), guard the KMP/Ktor action client against SSRF (B6 parity), and scope Android cleartext to loopback (drop the implicit blanket).

**Architecture:** Two layers. (1) `engines/kmp-engine` (pure Kotlin, **fully testable here** via `:jvmTest`) gets the SCRIPT-gate ctor flag + a Kotlin port of W3's `server-uri.ts` SSRF guard, applied in the validator and the `KtorActionInvoker` chokepoint. (2) `engines/android-app` (Android module; testable here via `:app:testDebugUnitTest` since the SDK is present + the module has been built before) gets the `FileProcessor` Zip-Slip containment, a scoped `network_security_config.xml`, and the SCRIPT opt-in UI wiring. The engine SCRIPT flag **defaults off**, so the device is fail-safe regardless of UI state.

**Tech Stack:** Kotlin Multiplatform (commonMain/jvmMain), Ktor client (`io.ktor.http.Url` is a commonMain dep — reused for URL parsing), Rhino (jvmMain SCRIPT exec), `kotlin.test` (commonTest) + JUnit5/Jackson (jvmTest); Android: Compose, JUnit4 + `TemporaryFolder` (unit tests), `network-security-config` XML.

**Repo/branch:** `TrajectoryRuntime`, new branch `hardening/android` off `hardening/runtime-untrusted` (W3, stacked). PR **base = `hardening/runtime-untrusted`** (merge order W1 #2 → W3 #3 → W4). No file overlap with W3 (W3 = web/web-ui; W4 = kmp-engine + android-app), so no conflicts.

**Test runners:**
- KMP: `cd engines/kmp-engine && .\gradlew.bat :jvmTest` (JUnit5 platform; commonTest `kotlin.test` compiles into the jvm test set). One class: append `--tests` is not reliable for KMP DynamicTest; run the whole `:jvmTest`.
- Android: `cd engines/android-app && .\gradlew.bat :app:testDebugUnitTest` (JUnit4 unit tests). Manifest/network-config validated by `.\gradlew.bat :app:processDebugManifest` (or `:app:assembleDebug`). **Requires `local.properties` with `sdk.dir` (T0).** If Gradle cannot resolve dependencies offline, fall back to careful review + flag for the user to run the Android verification; the KMP layer is the security boundary and is fully verified here regardless.

**Commit trailer:** every commit ends with `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`. (Runtime has no husky.)

**Scope note (locked spec):** Per the design spec §2.3, SCRIPT mitigation is **off-by-default only — NO Rhino `ClassShutter` / sandbox engineering** (the HANDOFF's "+ ClassShutter" is superseded). `ScriptExecutor.jvm.kt` (Rhino) is therefore **not modified**; the gate lives in `WorkflowEngine`, mirroring W3-web's `engine.ts` gate.

---

## T0 — Branch + Android SDK pointer

- [ ] **Step 1: Create branch**

Run: `cd C:/Trajectory/TrajectoryRuntime && git switch -c hardening/android hardening/runtime-untrusted`

- [ ] **Step 2: Point Gradle at the Android SDK (gitignored)**

The SDK is at `%LOCALAPPDATA%\Android\Sdk` but there is no `local.properties`. Create one (forward-slash or escaped path):

`engines/android-app/local.properties`:
```properties
sdk.dir=C\:\\Users\\dnbra\\AppData\\Local\\Android\\Sdk
```

- [ ] **Step 3: Confirm it is ignored, not tracked**

Run: `git -C C:/Trajectory/TrajectoryRuntime check-ignore engines/android-app/local.properties`
Expected: prints the path (ignored). If it prints nothing, add `local.properties` to `engines/android-app/.gitignore` and re-check. **Never commit `local.properties`.**

---

## T1 — B6 SSRF: Kotlin port of the server-URI guard (commonMain)

**Files:**
- Create: `engines/kmp-engine/src/commonMain/kotlin/com/trajectoryruntime/engine/ServerUri.kt`
- Test: `engines/kmp-engine/src/commonTest/kotlin/com/trajectoryruntime/engine/ServerUriTest.kt`

This mirrors W3's `engines/web/src/lib/server-uri.ts` (http(s)-only, loopback-by-default, optional allow-list).

- [ ] **Step 1: Write the failing test**

`ServerUriTest.kt`:
```kotlin
// Copyright (c) 2026 Dennis Brandl
// Licensed under the Apache License, Version 2.0. See LICENSE for details.
package com.trajectoryruntime.engine

import kotlin.test.Test
import kotlin.test.assertFailsWith
import kotlin.test.assertFalse
import kotlin.test.assertTrue

class ServerUriTest {
    @Test fun loopbackAllowedByDefault() {
        assertTrue(isAllowedServerUri("http://localhost:3002"))
        assertTrue(isAllowedServerUri("http://127.0.0.1:3002/"))
        assertTrue(isAllowedServerUri("http://[::1]:3002"))
    }

    @Test fun externalAndPrivateRejectedByDefault() {
        assertFalse(isAllowedServerUri("https://evil.example"))
        assertFalse(isAllowedServerUri("http://169.254.169.254/"))   // cloud metadata
        assertFalse(isAllowedServerUri("http://10.0.0.5:8080"))      // private LAN
    }

    @Test fun nonHttpSchemeRejected() {
        assertFalse(isAllowedServerUri("file:///etc/passwd"))
        assertFalse(hasValidServerUriScheme("file:///etc/passwd"))
        assertTrue(hasValidServerUriScheme("http://anything"))
        assertTrue(hasValidServerUriScheme("https://anything"))
    }

    @Test fun allowlistHonored() {
        val allow = listOf("https://actions.example.com")
        assertTrue(isAllowedServerUri("https://actions.example.com/v1/", allow))
        assertFalse(isAllowedServerUri("https://other.example.com", allow))
    }

    @Test fun assertThrowsOnDisallowed() {
        assertFailsWith<DisallowedServerUriException> { assertAllowedServerUri("https://evil.example") }
        assertAllowedServerUri("http://localhost:3002") // must NOT throw
    }
}
```

- [ ] **Step 2: Run to verify it fails**

Run: `cd engines/kmp-engine && .\gradlew.bat :jvmTest`
Expected: compile failure (`ServerUri.kt` symbols undefined).

- [ ] **Step 3: Write the implementation**

`ServerUri.kt`:
```kotlin
// Copyright (c) 2026 Dennis Brandl
// Licensed under the Apache License, Version 2.0. See LICENSE for details.

// SSRF guard for action-server URIs. A workflow package's
// action_server_specifications[].uri (and the per-step server binding) is
// author-controlled; the runtime fetches it (capabilities, invoke, command,
// delete), so it must be constrained. Kotlin port of engines/web/src/lib/server-uri.ts.
package com.trajectoryruntime.engine

import io.ktor.http.Url

class DisallowedServerUriException(message: String) : Exception(message)

private val LOOPBACK_V4 = Regex("^127(?:\\.\\d{1,3}){3}$")

private fun isLoopbackHost(hostname: String): Boolean {
    val h = hostname.trim('[', ']') // strip IPv6 brackets if present
    return h == "localhost" || h == "::1" || LOOPBACK_V4.matches(h)
}

private fun schemeOf(uri: String): String? =
    try { Url(uri).protocol.name.lowercase() } catch (_: Throwable) { null }

/** Lightweight parseable + http(s)-scheme check (used by the validator). */
fun hasValidServerUriScheme(uri: String): Boolean {
    val s = schemeOf(uri) ?: return false
    return s == "http" || s == "https"
}

/**
 * Whether the action-server URI may be contacted. http(s) only; loopback by
 * default (blocks external, private-LAN, and cloud-metadata hosts). If an
 * allow-list is supplied, the URI must match one of its origins/prefixes.
 */
fun isAllowedServerUri(uri: String, allowlist: List<String> = emptyList()): Boolean {
    val url = try { Url(uri) } catch (_: Throwable) { return false }
    val scheme = url.protocol.name.lowercase()
    if (scheme != "http" && scheme != "https") return false
    val list = allowlist.map { it.trim() }.filter { it.isNotEmpty() }
    if (list.isNotEmpty()) {
        val origin = buildString {
            append(scheme).append("://").append(url.host)
            if (url.port != url.protocol.defaultPort && url.port != 0) append(":").append(url.port)
        }
        return list.any { origin == it || uri.startsWith(it) }
    }
    return isLoopbackHost(url.host)
}

fun assertAllowedServerUri(uri: String, allowlist: List<String> = emptyList()) {
    if (!isAllowedServerUri(uri, allowlist)) {
        throw DisallowedServerUriException(
            "Action-server URI is not allowed: \"$uri\". It must be http(s) and loopback, " +
                "or listed in the trusted action-server allow-list.",
        )
    }
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `cd engines/kmp-engine && .\gradlew.bat :jvmTest`
Expected: PASS (all `ServerUriTest` cases green; existing suite unaffected).

- [ ] **Step 5: Commit**

```bash
git add engines/kmp-engine/src/commonMain/kotlin/com/trajectoryruntime/engine/ServerUri.kt engines/kmp-engine/src/commonTest/kotlin/com/trajectoryruntime/engine/ServerUriTest.kt
git commit -m "fix(kmp): SSRF guard for action-server URIs (scheme + loopback/allowlist)"
```

---

## T2 — B6 SSRF: validator scheme phase (commonMain)

**Files:**
- Modify: `engines/kmp-engine/src/commonMain/kotlin/com/trajectoryruntime/engine/Validator.kt`
- Test: `engines/kmp-engine/src/jvmTest/kotlin/com/trajectoryruntime/engine/ActionServerUriValidatorTest.kt`

Mirrors web `validator.ts` `actionServerUriValidation` (Phase A2.5). No existing conformance fixture defines `action_server_specifications`, so this is a no-op for the conformance suite.

- [ ] **Step 1: Write the failing test**

`ActionServerUriValidatorTest.kt`:
```kotlin
// Copyright (c) 2026 Dennis Brandl
// Licensed under the Apache License, Version 2.0. See LICENSE for details.
package com.trajectoryruntime.engine

import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class ActionServerUriValidatorTest {
    private fun parseWorkflow(json: String): Map<String, Any?> {
        @Suppress("UNCHECKED_CAST")
        return com.fasterxml.jackson.databind.ObjectMapper().readValue(json, Map::class.java) as Map<String, Any?>
    }

    private fun wf(serverUri: String) = """{
      "local_id":"wf","oid":"wf","version":"1.0.0","last_modified_date":"2026-06-04",
      "steps":[
        {"local_id":"start","oid":"s1","version":"1.0.0","last_modified_date":"2026-06-04","step_type":"START"},
        {"local_id":"end","oid":"s2","version":"1.0.0","last_modified_date":"2026-06-04","step_type":"END"}
      ],
      "connections":[{"from_step_id":"s1","to_step_id":"s2"}],
      "environment_specifications":[
        {"local_id":"env","oid":"env-1","version":"1.0.0","last_modified_date":"2026-06-04",
         "action_server_specifications":[{"uri":"$serverUri"}]}
      ]
    }""".trimIndent()

    @Test fun `valid http server uri passes`() {
        val r = validate(parseWorkflow(wf("http://localhost:3002")))
        assertTrue(r.valid, "expected valid, got ${r.error_code}: ${r.error_message}")
    }

    @Test fun `non-http server uri is rejected`() {
        val r = validate(parseWorkflow(wf("file:///etc/passwd")))
        assertEquals(false, r.valid)
        assertEquals("INVALID_VALIDATION", r.error_code)
    }
}
```

- [ ] **Step 2: Run to verify it fails**

Run: `cd engines/kmp-engine && .\gradlew.bat :jvmTest`
Expected: `non-http server uri is rejected` FAILS (today the bad URI is accepted → `valid=true`).

- [ ] **Step 3: Write the implementation**

In `Validator.kt`, add this function immediately after `actionProxyValidation` (before `tryCatchValidation`):
```kotlin
private fun actionServerUriValidation(workflow: Map<String, Any?>): ValidationResult? {
    @Suppress("UNCHECKED_CAST")
    val envSpecs = workflow["environment_specifications"] as? List<Map<String, Any?>> ?: return null
    for (env in envSpecs) {
        @Suppress("UNCHECKED_CAST")
        val servers = env["action_server_specifications"] as? List<Map<String, Any?>> ?: continue
        for (s in servers) {
            val uri = s["uri"]
            if (uri !is String || !hasValidServerUriScheme(uri)) {
                return ValidationResult(
                    false,
                    "INVALID_VALIDATION",
                    "Action server URI is not a valid http(s) URL: $uri",
                )
            }
        }
    }
    return null
}
```
Then in `fun validate(...)`, add the phase call after the ACTION PROXY phase (after the `actionProxyValidation(workflow)?.let { return it }` line, before the final `return ValidationResult(valid = true)`):
```kotlin
    // Phase C.5: action-server URI scheme check (SSRF — reject non-http(s) URIs)
    actionServerUriValidation(workflow)?.let { return it }
```

- [ ] **Step 4: Run to verify it passes**

Run: `cd engines/kmp-engine && .\gradlew.bat :jvmTest`
Expected: PASS (both new cases + the full conformance/validator suite stay green).

- [ ] **Step 5: Commit**

```bash
git add engines/kmp-engine/src/commonMain/kotlin/com/trajectoryruntime/engine/Validator.kt engines/kmp-engine/src/jvmTest/kotlin/com/trajectoryruntime/engine/ActionServerUriValidatorTest.kt
git commit -m "fix(kmp): validator rejects non-http(s) action-server URIs (SSRF scheme check)"
```

---

## T3 — B6 SSRF: guard the KtorActionInvoker chokepoint (commonMain)

**Files:**
- Modify: `engines/kmp-engine/src/commonMain/kotlin/com/trajectoryruntime/engine/KtorActionInvoker.kt`
- Test: `engines/kmp-engine/src/jvmTest/kotlin/com/trajectoryruntime/engine/KtorActionInvokerSsrfTest.kt`

`KtorActionInvoker` is defined but not yet instantiated anywhere in the repo, so adding a defaulted ctor param is safe. It is the network egress for ACTION PROXY on native platforms; guarding it makes the SSRF policy enforced at the chokepoint (defense in depth with the validator).

- [ ] **Step 1: Write the failing test**

`KtorActionInvokerSsrfTest.kt`:
```kotlin
// Copyright (c) 2026 Dennis Brandl
// Licensed under the Apache License, Version 2.0. See LICENSE for details.
package com.trajectoryruntime.engine

import kotlinx.coroutines.runBlocking
import kotlin.test.Test
import kotlin.test.assertFailsWith

class KtorActionInvokerSsrfTest {
    private val noopCallbacks = object : ActionInvokerCallbacks {
        override fun onStateChange(stepOid: String, newState: StepState, outputs: Map<String, String>?) {}
        override fun onConnectivityChange(stepOid: String, status: String) {}
    }

    @Test fun `invoke rejects a non-loopback server uri before any network call`() {
        val invoker = KtorActionInvoker()
        val req = InvokeRequestKmp(
            stepOid = "s", workflowInstanceId = "w",
            serverUri = "http://169.254.169.254", // cloud-metadata SSRF target
            actionOid = "a", inputs = emptyMap(), pollIntervalMs = 4000,
        )
        assertFailsWith<DisallowedServerUriException> {
            runBlocking { invoker.invoke(req, noopCallbacks) }
        }
    }
}
```

- [ ] **Step 2: Run to verify it fails**

Run: `cd engines/kmp-engine && .\gradlew.bat :jvmTest`
Expected: FAIL — today `invoke` attempts the request (no guard); the test expects an immediate `DisallowedServerUriException`.

- [ ] **Step 3: Write the implementation**

In `KtorActionInvoker.kt`:
1. Add an allow-list ctor param:
```kotlin
class KtorActionInvoker(
    private val allowlist: List<String> = emptyList(),
) : ActionInvoker {
```
2. Guard `invoke` — add as the **first** line of the method body (before `val base = normalize(req.serverUri)`):
```kotlin
        assertAllowedServerUri(req.serverUri, allowlist)
```
3. Guard `sendCommand` and `abort` — add as the first line of each, **before** their `try` blocks (so a disallowed URI throws rather than being swallowed as best-effort):
```kotlin
        assertAllowedServerUri(serverUri, allowlist)
```

- [ ] **Step 4: Run to verify it passes**

Run: `cd engines/kmp-engine && .\gradlew.bat :jvmTest`
Expected: PASS. (`invoke` calls the guard before constructing the request; the metadata IP is non-loopback → throws.)

- [ ] **Step 5: Commit**

```bash
git add engines/kmp-engine/src/commonMain/kotlin/com/trajectoryruntime/engine/KtorActionInvoker.kt engines/kmp-engine/src/jvmTest/kotlin/com/trajectoryruntime/engine/KtorActionInvokerSsrfTest.kt
git commit -m "fix(kmp): enforce SSRF allow-list in KtorActionInvoker (invoke/command/abort)"
```

---

## T4 — B4 SCRIPT off-by-default in the KMP engine (commonMain)

**Files:**
- Modify: `engines/kmp-engine/src/commonMain/kotlin/com/trajectoryruntime/engine/WorkflowEngine.kt` (ctor + SCRIPT dispatch + child-engine propagation)
- Modify: `engines/kmp-engine/src/jvmTest/kotlin/com/trajectoryruntime/engine/ConformanceRunner.kt` (pass the flag for trusted fixtures)
- Test: `engines/kmp-engine/src/commonTest/kotlin/com/trajectoryruntime/engine/ScriptGateKmpTest.kt`

Mirrors W3-web `engine.ts` (flag default false; SCRIPT with a `source` ERRORs the workflow when disabled). **Conformance must pass `allowScriptExecution = true`** — `exec-child-002` executes `output.result = msg + '-processed'` and `exec-parallel-002/005` carry (comment) `source` values, so all three would ERROR if the gate defaulted off in the runner.

- [ ] **Step 1: Write the failing test**

`ScriptGateKmpTest.kt`:
```kotlin
// Copyright (c) 2026 Dennis Brandl
// Licensed under the Apache License, Version 2.0. See LICENSE for details.
package com.trajectoryruntime.engine

import kotlinx.serialization.json.Json
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertNull
import kotlin.test.assertTrue

class ScriptGateKmpTest {
    private val wfJson = """
    {
      "local_id":"wf","oid":"wf","version":"1.0","last_modified_date":"2026-06-04",
      "steps":[
        {"local_id":"start","oid":"start","version":"1.0","last_modified_date":"2026-06-04","step_type":"START"},
        {"local_id":"script","oid":"script","version":"1.0","last_modified_date":"2026-06-04","step_type":"SCRIPT",
         "script_config":{"language":"javascript","source":"output.Result = 'ran';"},
         "output_parameter_specifications":[{"id":"Result","oid":"op1","target":"Out.Result"}]},
        {"local_id":"end","oid":"end","version":"1.0","last_modified_date":"2026-06-04","step_type":"END"}
      ],
      "connections":[
        {"from_step_id":"start","to_step_id":"script"},
        {"from_step_id":"script","to_step_id":"end"}
      ]
    }
    """.trimIndent()

    private fun wf() = Json { ignoreUnknownKeys = true }.decodeFromString<MasterWorkflowSpecification>(wfJson)

    @Test fun scriptIsDisabledByDefault() {
        val engine = WorkflowEngine(wf())
        engine.start()
        assertEquals(WorkflowState.ERRORED, engine.getWorkflowState())
        assertNull(engine.getProperties()["Out.Result"])
        val errored = engine.getTrace().firstOrNull { it.state == "ERRORED" }
        assertTrue(errored?.error?.contains("disabled", ignoreCase = true) == true, "expected a 'disabled' error trace")
    }

    @Test fun scriptExecutesWhenAllowed() {
        val engine = WorkflowEngine(wf(), allowScriptExecution = true)
        engine.start()
        assertEquals(WorkflowState.COMPLETED, engine.getWorkflowState())
        assertEquals("ran", engine.getProperties()["Out.Result"])
    }
}
```

- [ ] **Step 2: Run to verify it fails**

Run: `cd engines/kmp-engine && .\gradlew.bat :jvmTest`
Expected: compile failure (`allowScriptExecution` ctor arg does not exist).

- [ ] **Step 3: Implement the ctor flag**

In `WorkflowEngine.kt`, change the constructor to add a 4th parameter:
```kotlin
class WorkflowEngine(
    private val workflow: MasterWorkflowSpecification,
    setup: TestFixtureSetup? = null,
    resourceManager: ResourceManager? = null,
    private val allowScriptExecution: Boolean = false,
) {
```

- [ ] **Step 4: Implement the SCRIPT gate**

In `activateStepAfterResources`, inside `if (target.stepType == "SCRIPT") {`, insert this as the **first** statement (before `val inputParams = ...`):
```kotlin
                if (target.step.script_config?.source != null && !allowScriptExecution) {
                    recordTrace(
                        target.oid,
                        "ERRORED",
                        error = "SCRIPT execution is disabled — enable it only for trusted packages",
                    )
                    target.state = StepState.ERRORED
                    workflowState = WorkflowState.ERRORED
                    val known = mutableSetOf<String>()
                    collectKnownStepOids(known)
                    resourceManager?.cancelQueuedWaiters(known)
                    return
                }
```

- [ ] **Step 5: Propagate the flag to child engines**

In `activateWorkflowProxy`, the child-engine construction currently reads:
```kotlin
        val childEngine = WorkflowEngine(childSpec, TestFixtureSetup(
            starting_parameters = startingParams,
            initial_properties = initialProperties,
        ), resourceManager)
```
Change the last argument list to also pass the flag:
```kotlin
        val childEngine = WorkflowEngine(childSpec, TestFixtureSetup(
            starting_parameters = startingParams,
            initial_properties = initialProperties,
        ), resourceManager, allowScriptExecution)
```

- [ ] **Step 6: Keep conformance trusted — pass the flag in ConformanceRunner**

In `ConformanceRunner.kt` `runExecutionFixture`, change:
```kotlin
        val engine = WorkflowEngine(workflow, fixture.setup)
```
to:
```kotlin
        // Conformance fixtures are trusted spec contracts; SCRIPT steps must execute.
        val engine = WorkflowEngine(workflow, fixture.setup, allowScriptExecution = true)
```

- [ ] **Step 7: Run to verify it passes**

Run: `cd engines/kmp-engine && .\gradlew.bat :jvmTest`
Expected: PASS — `ScriptGateKmpTest` green AND the full conformance suite (incl. `exec-child-002`, `exec-parallel-002`, `exec-parallel-005`) stays green because the runner now opts in.

- [ ] **Step 8: Commit**

```bash
git add engines/kmp-engine/src/commonMain/kotlin/com/trajectoryruntime/engine/WorkflowEngine.kt engines/kmp-engine/src/jvmTest/kotlin/com/trajectoryruntime/engine/ConformanceRunner.kt engines/kmp-engine/src/commonTest/kotlin/com/trajectoryruntime/engine/ScriptGateKmpTest.kt
git commit -m "feat(kmp): SCRIPT execution off by default; opt-in via WorkflowEngine flag"
```

---

## T5 — B7 Zip-Slip containment in FileProcessor (android-app)

**Files:**
- Modify: `engines/android-app/app/src/main/kotlin/io/saturnis/trajectory/util/FileProcessor.kt` (`extractZip`, ~:58-113)
- Test: `engines/android-app/app/src/test/kotlin/io/saturnis/trajectory/util/FileProcessorTest.kt` (add one test)

`extractZip` writes `File(outputDir, entry.name)` with no containment — a `../` or absolute entry escapes `outputDir` (Zip Slip). `processFile` delegates to `extractZip`, so fixing it here covers both paths.

- [ ] **Step 1: Write the failing test**

Append to `FileProcessorTest.kt` (inside the class; `assertThrows` is available via `import org.junit.Assert.*` with junit 4.13.2):
```kotlin
    @Test
    fun `extractZip rejects path-traversal entries (Zip Slip)`() {
        val zipFile = tempDir.newFile("evil.WFmasterX")
        ZipOutputStream(zipFile.outputStream()).use { zos ->
            zos.putNextEntry(ZipEntry("Test Workflow.WFmaster"))
            zos.write("""{"local_id":"wf-1","oid":"oid-1","version":"1.0","last_modified_date":"2026-01-01","steps":[],"connections":[]}""".toByteArray())
            zos.closeEntry()
            zos.putNextEntry(ZipEntry("../../evil.txt"))
            zos.write("pwned".toByteArray())
            zos.closeEntry()
        }
        val outputDir = tempDir.newFolder("output")
        assertThrows(SecurityException::class.java) {
            FileProcessor.extractZip(zipFile, outputDir)
        }
        // The escaping entry must never be written outside outputDir.
        assertFalse(File(outputDir.parentFile, "evil.txt").exists())
    }
```
Add the import at the top if not present: `import java.io.File`.

- [ ] **Step 2: Run to verify it fails**

Run: `cd engines/android-app && .\gradlew.bat :app:testDebugUnitTest`
Expected: FAIL — today the `../../evil.txt` entry is written (no exception). *(If Gradle cannot resolve dependencies offline, record the limitation and verify by review; proceed to implement.)*

- [ ] **Step 3: Write the implementation**

In `extractZip`, compute the canonical output root once before the loop, and assert containment before writing. Replace:
```kotlin
        ZipInputStream(zipFile.inputStream()).use { zis ->
            var entry = zis.nextEntry
            while (entry != null) {
                if (!entry.isDirectory) {
                    val name = entry.name
                    val outFile = File(outputDir, name)
                    outFile.parentFile?.mkdirs()
                    outFile.outputStream().use { zis.copyTo(it) }
```
with:
```kotlin
        val outputRoot = outputDir.canonicalFile
        ZipInputStream(zipFile.inputStream()).use { zis ->
            var entry = zis.nextEntry
            while (entry != null) {
                if (!entry.isDirectory) {
                    val name = entry.name
                    val outFile = File(outputDir, name)
                    val canonical = outFile.canonicalFile
                    // Zip-Slip containment: the resolved path must stay under outputDir.
                    if (canonical != outputRoot &&
                        !canonical.path.startsWith(outputRoot.path + File.separator)
                    ) {
                        throw SecurityException("Zip entry escapes target directory: $name")
                    }
                    outFile.parentFile?.mkdirs()
                    outFile.outputStream().use { zis.copyTo(it) }
```
(The rest of the `when {…}` block and loop are unchanged.)

- [ ] **Step 4: Run to verify it passes**

Run: `cd engines/android-app && .\gradlew.bat :app:testDebugUnitTest`
Expected: PASS — new Zip-Slip test green; existing `extractZip`/`detectFormat` tests still green. *(If un-buildable here: confirm by review that the containment check precedes every write and that absolute/`..` entries resolve outside `outputRoot`; flag for the user to run.)*

- [ ] **Step 5: Commit**

```bash
git add engines/android-app/app/src/main/kotlin/io/saturnis/trajectory/util/FileProcessor.kt engines/android-app/app/src/test/kotlin/io/saturnis/trajectory/util/FileProcessorTest.kt
git commit -m "fix(android): reject Zip-Slip path-traversal entries in FileProcessor.extractZip"
```

---

## T6 — Scoped cleartext network-security config (android-app)

**Files:**
- Create: `engines/android-app/app/src/main/res/xml/network_security_config.xml`
- Modify: `engines/android-app/app/src/main/AndroidManifest.xml` (`<application>` attribute)

`minSdk = 29` means cleartext is platform-denied by default and there is no blanket `usesCleartextTraffic`. Make the policy **explicit and scoped**: deny cleartext globally, permit it only for loopback / emulator-loopback so the local self-host Action Container works while remote servers must use HTTPS (coherent with the SSRF loopback default).

- [ ] **Step 1: Create the network-security config**

`network_security_config.xml`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- Default: HTTPS only. Untrusted/remote action servers may not use cleartext. -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
    <!-- Scoped exception: a locally self-hosted Action Container reached over loopback
         (device localhost / 127.0.0.1, or the Android emulator host alias 10.0.2.2). -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="false">localhost</domain>
        <domain includeSubdomains="false">127.0.0.1</domain>
        <domain includeSubdomains="false">10.0.2.2</domain>
    </domain-config>
</network-security-config>
```

- [ ] **Step 2: Reference it from the manifest**

In `AndroidManifest.xml`, add the attribute to the `<application>` tag (e.g., after `android:label`):
```xml
        android:networkSecurityConfig="@xml/network_security_config"
```

- [ ] **Step 3: Verify the manifest/resource compiles**

Run: `cd engines/android-app && .\gradlew.bat :app:processDebugManifest`
Expected: success; merged manifest includes `android:networkSecurityConfig`. *(If un-buildable here: validate XML by review against the Android network-security-config schema; flag for the user.)*

- [ ] **Step 4: Commit**

```bash
git add engines/android-app/app/src/main/res/xml/network_security_config.xml engines/android-app/app/src/main/AndroidManifest.xml
git commit -m "fix(android): scoped cleartext network-security config (loopback only)"
```

---

## T7 — SCRIPT opt-in wiring on device (android-app)

**Files:**
- Modify: `engines/android-app/app/src/main/kotlin/io/saturnis/trajectory/coordinator/WorkflowCoordinator.kt`
- Modify: `engines/android-app/app/src/main/kotlin/io/saturnis/trajectory/manager/WorkflowManager.kt`
- Modify: `engines/android-app/app/src/main/kotlin/io/saturnis/trajectory/ui/screens/SettingsScreen.kt`
- Modify: the workflow-start call-site that has a `Context`/prefs (resolve during execution — the screen/VM that invokes the manager's start method; persisted-pref key `allow_script_execution`).

The engine default (T4) already guarantees SCRIPT is off on device. This task adds the **opt-in path**: a Settings toggle persisted to `trajectoryruntime_settings`/`allow_script_execution`, threaded into `loadAndStart`.

- [ ] **Step 1: Thread the flag through the coordinator**

In `WorkflowCoordinator.kt`:
1. Add a field near the other `private var _…` fields:
```kotlin
    private var _allowScriptExecution: Boolean = false
```
2. Add the parameter to `loadAndStart` (default false keeps all existing callers safe):
```kotlin
    fun loadAndStart(
        spec: MasterWorkflowSpecification,
        setup: TestFixtureSetup? = null,
        mediaMap: Map<String, String> = emptyMap(),
        environmentJsons: List<String> = emptyList(),
        sharedResourceManager: InMemoryResourceManager? = null,
        allowScriptExecution: Boolean = false,
    ) {
```
3. Store it (alongside the other `this._… =` assignments at the top of the body):
```kotlin
        this._allowScriptExecution = allowScriptExecution
```
4. Pass it to the engine — change `engine = WorkflowEngine(spec, mergedSetup, resourceManager)` to:
```kotlin
        engine = WorkflowEngine(spec, mergedSetup, resourceManager, allowScriptExecution)
```
5. Preserve it on `restart()` — change the `loadAndStart(...)` call to include `allowScriptExecution = _allowScriptExecution`.

- [ ] **Step 2: Thread the flag through the manager**

In `WorkflowManager.kt`, locate the public start method whose body contains the `coordinator.loadAndStart(loaded.spec, …)` call (~:169). Add `allowScriptExecution: Boolean = false` to that method's signature and pass it through:
```kotlin
        coordinator.loadAndStart(loaded.spec, setup = setup, mediaMap = loaded.mediaMap, environmentJsons = loaded.environmentJsons, sharedResourceManager = sharedRM, allowScriptExecution = allowScriptExecution)
```
(The resume/restore path at ~:441 may keep the default `false`; resumed workflows do not re-run already-completed SCRIPT steps.)

- [ ] **Step 3: Read the pref at the start call-site**

At the UI call-site that invokes the manager's start method (it has `LocalContext`/an `Activity`), read the pref and pass it:
```kotlin
val allowScript = context
    .getSharedPreferences("trajectoryruntime_settings", android.content.Context.MODE_PRIVATE)
    .getBoolean("allow_script_execution", false)
// …pass allowScriptExecution = allowScript into the manager start call
```

- [ ] **Step 4: Add the Settings toggle**

In `SettingsScreen.kt`:
1. Add a key constant near the others:
```kotlin
private const val KEY_ALLOW_SCRIPT = "allow_script_execution"
```
2. Add state near the other `var …confirmDelete…` declarations:
```kotlin
    var allowScript by remember {
        mutableStateOf(prefs.getBoolean(KEY_ALLOW_SCRIPT, false))
    }
```
3. Add a "Security" section (place it after the Confirmations section's `HorizontalDivider()`):
```kotlin
            // Security section
            Text(
                text = "Security",
                style = MaterialTheme.typography.titleSmall,
                color = MaterialTheme.colorScheme.primary,
            )
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically,
            ) {
                Column(modifier = Modifier.weight(1f)) {
                    Text(
                        text = "Allow SCRIPT execution from imported workflows",
                        style = MaterialTheme.typography.bodyMedium,
                    )
                    Text(
                        text = "SCRIPT steps run author-supplied JavaScript. Only enable for packages you trust.",
                        style = MaterialTheme.typography.bodySmall,
                        color = MaterialTheme.colorScheme.onSurfaceVariant,
                    )
                }
                Switch(
                    checked = allowScript,
                    onCheckedChange = { checked ->
                        allowScript = checked
                        prefs.edit().putBoolean(KEY_ALLOW_SCRIPT, checked).apply()
                    },
                )
            }
            HorizontalDivider()
```

- [ ] **Step 5: Verify it compiles**

Run: `cd engines/android-app && .\gradlew.bat :app:assembleDebug`
Expected: build succeeds. *(If un-buildable here: review that every `loadAndStart` caller still compiles with the defaulted param and the toggle reads/writes the correct key; flag for the user to build + smoke-test.)*

- [ ] **Step 6: Commit**

```bash
git add engines/android-app/app/src/main/kotlin/io/saturnis/trajectory/coordinator/WorkflowCoordinator.kt engines/android-app/app/src/main/kotlin/io/saturnis/trajectory/manager/WorkflowManager.kt engines/android-app/app/src/main/kotlin/io/saturnis/trajectory/ui/screens/SettingsScreen.kt
# plus the start call-site file resolved in Step 3
git commit -m "feat(android): SCRIPT opt-in setting; off by default, threaded into engine"
```

---

## T8 — Full verify + push + PR

- [ ] **Step 1: KMP suite green**

Run: `cd engines/kmp-engine && .\gradlew.bat :jvmTest`
Expected: BUILD SUCCESSFUL; all conformance + new SSRF/SCRIPT tests pass.

- [ ] **Step 2: Android unit tests + assemble (best-effort here)**

Run: `cd engines/android-app && .\gradlew.bat :app:testDebugUnitTest :app:assembleDebug`
Expected: SUCCESSFUL. If the box cannot resolve Android deps, capture the exact failure and note in the PR that Android build/test verification is pending the user's environment; the KMP layer is fully verified.

- [ ] **Step 3: Push + open PR**

```bash
git push -u origin hardening/android
gh pr create --base hardening/runtime-untrusted --head hardening/android \
  --title "W4: Android/KMP hardening (Zip-Slip, SCRIPT off-by-default, SSRF, scoped cleartext)" \
  --body "<summary of B7 / B4-parity / B6-parity / cleartext; note Android build status>"
```

- [ ] **Step 4: STOP for user review.** Do not merge. Report verification output (KMP green; Android status).

---

## Notes / risks
- **SCRIPT scope:** off-by-default only, **no Rhino `ClassShutter`** (locked spec §2.3). `ScriptExecutor.jvm.kt` is intentionally untouched.
- **Conformance parity:** the shared `spec/conformance` fixtures are executed **only** by the KMP `ConformanceRunner` (the web node:test suite does not load them). Hence the runner — not a web harness — is where `allowScriptExecution = true` must be set for trusted fixtures.
- **SSRF allow-list source:** `KtorActionInvoker(allowlist = emptyList())` defaults to loopback-only. The invoker is not yet instantiated in the app; when ACTION PROXY is wired on device/iOS, pass a user-configured allow-list (future work, out of W4 scope).
- **Cleartext trade-off:** remote/LAN action servers must use HTTPS on device; only loopback + `10.0.2.2` are cleartext-permitted. Documented as a demo limitation.
- **Android buildability:** SDK present + prior `build/` dir suggest `:app:*` tasks work here; if offline dep resolution fails, T5–T7 are review-verified and flagged. The KMP security boundary (T1–T4) is fully test-verified regardless.
- **Validator drift:** unlike the web (two forked validators), the KMP engine has a single `Validator.kt`; no duplicate to keep in lockstep here.

## Self-review
- **Spec coverage:** B7 Zip-Slip → T5; scoped cleartext → T6; SCRIPT off-by-default (Android/KMP) → T4 (+ opt-in UI T7); Ktor SSRF → T1–T3. All W4 spec bullets mapped.
- **Type/signature consistency:** `WorkflowEngine` 4th param `allowScriptExecution: Boolean = false` used identically in T4 (ctor, child-engine call, ConformanceRunner) and T7 (coordinator). `assertAllowedServerUri(uri, allowlist)` / `hasValidServerUriScheme(uri)` / `isAllowedServerUri(uri, allowlist)` / `DisallowedServerUriException` defined in T1, used unchanged in T2 (validator) and T3 (invoker). `KtorActionInvoker(allowlist=…)` defaulted → no existing caller breaks.
- **No placeholders:** every code step has complete code. Two execution-time lookups are explicitly bounded (T7 manager start-method name; T7 UI start call-site) and compile-guarded by the build; the security default is enforced in the engine ctor independent of that wiring.
