# TRY / CATCH / RETURN — Native (Kotlin) Parity + Conformance Capstone

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bring the Kotlin `kmp-engine` (native Android/iOS) to TRY/CATCH/RETURN parity with the TypeScript engine, then add the **shared `spec/conformance` fixtures** that prove both engines behave identically.

**Architecture:** Direct mirror of the web/TS plan (`2026-05-31-try-catch-runtime-web-ts.md`), in Kotlin. The Kotlin engine already mirrors the TS engine method-for-method (`recordTrace`, `activateStep`, `submitAction`, `abortWorkflow`, `restartToSteps`, `releaseAllResources`, `PropertyStore.initializeFromWorkflow`), and a Kotlin `validate()` already mirrors the TS validator. We add: the new types, validator rules (with the catch-network orphan exemption), CATCH/RETURN auto-completing handlers, a fail signal, TRY routing, RETURN dispatch, and a conformance-mode ACTION PROXY parking fix. The capstone fixtures live in `spec/conformance` and are auto-discovered by **both** engines' runners. Grounding map: `2026-05-31-try-catch-IMPLEMENTATION-NOTES.md`. Canonical behaviour: `specs/2026-05-31-try-catch-return-design.md`.

**Tech Stack:** Kotlin (Multiplatform, commonMain), kotlin.test on JUnit 5, Gradle 8.11.1, JDK 17/21. jvmTest also uses `kotlinx.serialization` + `jackson-databind`.

**Prerequisite:** The web/TS plan (`2026-05-31-try-catch-runtime-web-ts.md`) should be complete first — this plan's capstone fixtures are run against both engines, and the TS engine must already implement the feature for them to pass there.

**Scope boundaries:**
- Kotlin engine semantics + the shared conformance fixtures. `release_on_catch` is **descoped** (advisory only; no auto-release), same as the TS plan.
- The native **android-app coordinator** wiring (feeding real action failures into the engine, mirroring the web-ui coordinator) is **out of scope** here — this plan brings the *engine + conformance* to parity. Wiring the android-app `WorkflowCoordinator.kt`/invoker to the new fail signal is a follow-up (the same shape as web/TS Phase F).
- The JS facade (`JsApiFacade.kt`) needs no change: extending `@Serializable UserAction`/`TraceEntry` flows through automatically, and conformance runs on the JVM, not the JS facade.

---

## Test workflow (every task)

Run from `C:/Trajectory/TrajectoryRuntime/engines/kmp-engine`:
```
.\gradlew.bat :jvmTest
```
Run a single test class:
```
.\gradlew.bat :jvmTest --tests "com.trajectoryruntime.engine.TryCatchValidatorTest"
```
`:jvmTest` compiles `commonMain`+`jvmMain` then runs every `src/jvmTest` test, **including `ConformanceRunner`** (all four fixture categories). Add `--rerun-tasks` to force a re-run with no source change. Commit after each task with the message shown.

Tests calling `validate(...)` build the workflow as a JSON string and convert to `Map<String, Any?>` via Jackson (mirrors `ConformanceRunner.jsonObjectToMap`). Tests driving the engine deserialize the JSON to `MasterWorkflowSpecification` via kotlinx (mirrors `ConformanceRunner.runExecutionFixture`). Shared helpers (put at the top of each new test file):
```kotlin
import com.fasterxml.jackson.databind.ObjectMapper
import kotlinx.serialization.json.Json
private val MAPPER = ObjectMapper()
@Suppress("UNCHECKED_CAST")
fun wfMap(json: String): Map<String, Any?> = MAPPER.readValue(json, Map::class.java) as Map<String, Any?>
private val LENIENT = Json { ignoreUnknownKeys = true }
fun wfSpec(json: String): MasterWorkflowSpecification = LENIENT.decodeFromString(json)
```

---

## File Structure Plan

### Files to create
| File | Responsibility |
|---|---|
| `src/commonMain/kotlin/com/trajectoryruntime/engine/CatchNetworkPartition.kt` | Pure helper: `partitionCatchNetworks(steps, connections)` (BFS-from-CATCH). Shared by `Validator` + `WorkflowEngine`. |
| `src/jvmTest/kotlin/com/trajectoryruntime/engine/CatchNetworkPartitionTest.kt` | Unit tests for the partition helper. |
| `src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchValidatorTest.kt` | Validator rule tests (Phase K-B). |
| `src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchHandlersTest.kt` | `activateCatchStep` test (Phase K-C). |
| `src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchEngineTest.kt` | Engine integration tests: fail signal, TRY routing, RETURN dispatch (K-D, K-E). |
| `spec/conformance/validation/val-try-catch-*.json` | The 7 validation fixtures V1–V7 (capstone). |
| `spec/conformance/execution/exec-try-catch-*.json` | The execution fixtures (capstone; assert `workflow_state` + `final_properties`). |

### Files to modify
| File | Change |
|---|---|
| `src/commonMain/kotlin/com/trajectoryruntime/engine/Types.kt` | Add `FailureMode`/`TrySpecification`/`ReturnConfig`/`CatchContext`; extend `UserAction` with `failure_mode`/`error`; add `try_specifications`/`catch_id`/`return_config` to `MasterWorkflowStep`. |
| `src/commonMain/kotlin/com/trajectoryruntime/engine/Validator.kt` | Add `CATCH`/`RETURN` to `VALID_STEP_TYPES`; exempt catch networks from `ORPHANED_STEP`; add a `tryCatchValidation` phase. |
| `src/commonMain/kotlin/com/trajectoryruntime/engine/StepHandlers.kt` | Add `CATCH`/`RETURN` to `isAutoCompleting`; add `activateCatchStep`. |
| `src/commonMain/kotlin/com/trajectoryruntime/engine/WorkflowEngine.kt` | `active_catches` map; CATCH/RETURN dispatch in `activateStepAfterResources`; conformance-mode ACTION PROXY parking; `fail` branch in `submitAction`; RETURN command dispatch. |
| `spec/conformance/test-fixture-schema.json` | Add `ERRORED` to the `expected.workflow_state` enum. |

---

## Phase K-A — Types + step-type allow-list

### Task KA1: Kotlin types + VALID_STEP_TYPES + isAutoCompleting

**Files:** Modify `Types.kt`, `Validator.kt`, `StepHandlers.kt`. Create `TryCatchValidatorTest.kt`.

- [ ] **Step 1: Write the failing test** — create `src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchValidatorTest.kt`:
```kotlin
// Copyright (c) 2026 Saturnis.io. All rights reserved.
// Licensed under the GNU AGPL v3. See LICENSE.md for details.
package com.trajectoryruntime.engine

import com.fasterxml.jackson.databind.ObjectMapper
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertNotEquals
import kotlin.test.assertTrue

private val MAPPER = ObjectMapper()
@Suppress("UNCHECKED_CAST")
private fun wfMap(json: String): Map<String, Any?> = MAPPER.readValue(json, Map::class.java) as Map<String, Any?>

private const val DATE = "2026-05-31T12:00:00.000Z"

/** START→Action→END + a CATCH→RETURN island. Override pieces via the params. */
private fun baseWithCatch(trySpecJson: String = """[{"mode":"ERROR","catch_id":"C1"}]""",
                          returnJson: String = """{"command":"ABANDON"}""",
                          extra: String = ""): String = """
{
  "local_id":"wf","oid":"wf-1","version":"1.0.0","last_modified_date":"$DATE","schemaVersion":"4.0",
  "steps":[
    {"local_id":"Start","oid":"s1","version":"1.0.0","last_modified_date":"$DATE","step_type":"START"},
    {"local_id":"Action","oid":"s2","version":"1.0.0","last_modified_date":"$DATE","step_type":"ACTION PROXY","try_specifications":$trySpecJson},
    {"local_id":"End","oid":"s3","version":"1.0.0","last_modified_date":"$DATE","step_type":"END"},
    {"local_id":"Catch","oid":"c1","version":"1.0.0","last_modified_date":"$DATE","step_type":"CATCH","catch_id":"C1"},
    {"local_id":"Ret","oid":"r1","version":"1.0.0","last_modified_date":"$DATE","step_type":"RETURN","return_config":$returnJson}
    $extra
  ],
  "connections":[
    {"from_step_id":"s1","to_step_id":"s2"},
    {"from_step_id":"s2","to_step_id":"s3"},
    {"from_step_id":"c1","to_step_id":"r1"}
  ]
}"""

class TryCatchValidatorTest {
  @Test fun `CATCH and RETURN are valid step types`() {
    // Until the orphan exemption (KB2) the island is ORPHANED_STEP, but it must NOT be INVALID_STEP_TYPE.
    val r = validate(wfMap(baseWithCatch()))
    assertNotEquals("INVALID_STEP_TYPE", r.error_code, "CATCH/RETURN must be accepted step types")
  }
}
```

- [ ] **Step 2: Run — expect failure** (`.\gradlew.bat :jvmTest --tests "com.trajectoryruntime.engine.TryCatchValidatorTest"`): `validate` returns `INVALID_STEP_TYPE` (CATCH/RETURN not in `VALID_STEP_TYPES`).

- [ ] **Step 3: Implement**

In `Validator.kt`, add `"CATCH", "RETURN"` to `VALID_STEP_TYPES` (lines 177-182):
```kotlin
private val VALID_STEP_TYPES = setOf(
    "START", "END", "PARALLEL", "WAIT ALL", "WAIT ANY",
    "SELECT 1", "SELECT_1", "SCRIPT", "MATH",
    "USER_INTERACTION", "YES_NO", "WORKFLOW PROXY",
    "WAIT ACTION PROXY", "ACTION PROXY",
    "CATCH", "RETURN",
)
```
In `StepHandlers.kt`, add `CATCH`/`RETURN` to `isAutoCompleting` (lines 14-17):
```kotlin
fun isAutoCompleting(stepType: String): Boolean {
    val t = canonicalStepType(stepType)
    return t in listOf("START", "END", "PARALLEL", "WAIT ANY", "SELECT 1", "SCRIPT", "MATH", "CATCH", "RETURN")
}
```
In `Types.kt`, add the data classes (near `MasterWorkflowStep`) and extend `UserAction` + `MasterWorkflowStep`:
```kotlin
typealias FailureMode = String // "ERROR" | "ABORT" | "TIMEOUT"

@Serializable
data class TrySpecification(
    val mode: String,            // ERROR | ABORT | TIMEOUT
    val catch_id: String,
    val release_on_catch: Boolean? = null,
)
@Serializable
data class ReturnConfig(
    val command: String,         // ABANDON | RESTART | GOTO | RETRY
    val restart_mode: String? = null, // CLEAN | KEEP
    val goto_step_oid: String? = null,
)
/** Runtime-only (not serialized). */
data class CatchContext(
    val catch_oid: String,
    val trigger_step_oid: String,
    val trigger_step_name: String,
    val trigger_reason: String,
    val error_message: String?,
    val activated_at: String,
)
```
Extend `UserAction` (lines 342-348):
```kotlin
@Serializable
data class UserAction(
    val step_oid: String,
    val action: String, // "submit" | "button_press" | "yes" | "no" | "pause" | "resume" | "fail"
    val form_values: Map<String, JsonElement>? = null,
    val button_output: String? = null,
    val failure_mode: String? = null, // set when action == "fail": ERROR | ABORT | TIMEOUT
    val error: String? = null,        // set when action == "fail"
)
```
Extend `MasterWorkflowStep` (add to the data class, lines 147-165):
```kotlin
    val try_specifications: List<TrySpecification>? = null,
    val catch_id: String? = null,
    val return_config: ReturnConfig? = null,
```

- [ ] **Step 4: Run — expect pass.**

- [ ] **Step 5: Commit**
```
git add src/commonMain/kotlin/com/trajectoryruntime/engine/Types.kt src/commonMain/kotlin/com/trajectoryruntime/engine/Validator.kt src/commonMain/kotlin/com/trajectoryruntime/engine/StepHandlers.kt src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchValidatorTest.kt
git commit -m "feat(kmp): TRY/CATCH/RETURN types + step-type allow-list"
```

---

## Phase K-B — Validator rules

A new `tryCatchValidation` phase runs **after** `semanticValidation` and **before** `structuralValidation`, so its specific codes win. The orphan check in `semanticValidation` is exempted for catch-network steps.

### Task KB1: Partition helper

**Files:** Create `CatchNetworkPartition.kt` + `CatchNetworkPartitionTest.kt`.

- [ ] **Step 1: Write the failing test** — create `src/jvmTest/kotlin/com/trajectoryruntime/engine/CatchNetworkPartitionTest.kt`:
```kotlin
package com.trajectoryruntime.engine

import kotlin.test.Test
import kotlin.test.assertEquals

class CatchNetworkPartitionTest {
  private val steps = listOf(
    Triple("s1", "START", null), Triple("s2", "ACTION PROXY", null), Triple("s3", "END", null),
    Triple("c1", "CATCH", "C1"), Triple("c2", "USER_INTERACTION", null), Triple("c3", "RETURN", null),
  )
  private val conns = listOf("s1" to "s2", "s2" to "s3", "c1" to "c2", "c2" to "c3")

  @Test fun `separates main-flow from catch-network`() {
    val p = partitionCatchNetworks(steps, conns)
    assertEquals(setOf("s1", "s2", "s3"), p.mainFlowStepOids)
    assertEquals(setOf("c1", "c2", "c3"), p.catchNetworkStepOids)
    assertEquals(setOf("c1", "c2", "c3"), p.networksByCatchId["C1"])
  }

  @Test fun `excludes a CATCH with no reachable RETURN`() {
    val p = partitionCatchNetworks(
      listOf(Triple("s1","START",null), Triple("s2","END",null), Triple("c1","CATCH","C1"), Triple("c2","USER_INTERACTION",null)),
      listOf("s1" to "s2", "c1" to "c2"),
    )
    assertEquals(0, p.catchNetworkStepOids.size)
    assertEquals(4, p.mainFlowStepOids.size)
  }
}
```

- [ ] **Step 2: Run — expect failure** (unresolved reference `partitionCatchNetworks`).

- [ ] **Step 3: Implement** — create `src/commonMain/kotlin/com/trajectoryruntime/engine/CatchNetworkPartition.kt`:
```kotlin
// Copyright (c) 2026 Saturnis.io. All rights reserved.
// Licensed under the GNU AGPL v3. See LICENSE.md for details.
package com.trajectoryruntime.engine

data class CatchNetworkPartition(
    val mainFlowStepOids: Set<String>,
    val catchNetworkStepOids: Set<String>,
    /** catch_id -> set of step oids reachable from that CATCH (only if the subgraph reaches a RETURN). */
    val networksByCatchId: Map<String, Set<String>>,
)

/** Spec §6.5: a catch network is the BFS-closure from a CATCH over outgoing edges, counted only if it reaches a RETURN. */
fun partitionCatchNetworks(
    steps: List<Triple<String, String, String?>>, // (oid, stepType, catchId)
    connections: List<Pair<String, String>>,      // (fromOid, toOid)
): CatchNetworkPartition {
    val outEdges = HashMap<String, MutableList<String>>()
    for ((from, to) in connections) outEdges.getOrPut(from) { mutableListOf() }.add(to)
    val typeByOid = steps.associate { it.first to it.second }

    val networksByCatchId = HashMap<String, Set<String>>()
    val catchNetworkStepOids = HashSet<String>()
    for ((oid, type, catchId) in steps) {
        if (type != "CATCH" || catchId == null) continue
        val visited = HashSet<String>()
        val queue = ArrayDeque<String>()
        queue.add(oid)
        var hasReturn = false
        while (queue.isNotEmpty()) {
            val cur = queue.removeFirst()
            if (!visited.add(cur)) continue
            if (typeByOid[cur] == "RETURN") hasReturn = true
            for (n in outEdges[cur] ?: emptyList()) if (n !in visited) queue.add(n)
        }
        if (hasReturn) {
            networksByCatchId[catchId] = visited
            catchNetworkStepOids.addAll(visited)
        }
    }
    val mainFlowStepOids = steps.map { it.first }.filterNot { it in catchNetworkStepOids }.toSet()
    return CatchNetworkPartition(mainFlowStepOids, catchNetworkStepOids, networksByCatchId)
}
```

- [ ] **Step 4: Run — expect pass.** **Step 5: Commit**
```
git add src/commonMain/kotlin/com/trajectoryruntime/engine/CatchNetworkPartition.kt src/jvmTest/kotlin/com/trajectoryruntime/engine/CatchNetworkPartitionTest.kt
git commit -m "feat(kmp): catch-network partition helper"
```

### Task KB2: Orphan exemption (minimal valid workflow)

**Files:** Modify `Validator.kt`; modify `TryCatchValidatorTest.kt`.

- [ ] **Step 1: Add failing test** — append to `TryCatchValidatorTest`:
```kotlin
  @Test fun `accepts a minimal well-formed TRY CATCH RETURN workflow`() {
    val r = validate(wfMap(baseWithCatch()))
    assertEquals(true, r.valid, "expected valid, got ${r.error_code}: ${r.error_message}")
  }
```

- [ ] **Step 2: Run — expect failure** (`ORPHANED_STEP`).

- [ ] **Step 3: Implement** — in `Validator.kt` `semanticValidation`, replace the orphan loop (the `for (step in steps) { ... ORPHANED_STEP ... }` block) with a partition-exempted version:
```kotlin
    // Catch networks are intentional disconnected islands (entered at runtime via TRY, not a connection).
    val partition = partitionCatchNetworks(
        steps.map { Triple(it["oid"] as String, it["step_type"] as String, it["catch_id"] as String?) },
        connections.map { (it["from_step_id"] as String) to (it["to_step_id"] as String) },
    )
    for (step in steps) {
        val oid = step["oid"] as String
        if (oid in partition.catchNetworkStepOids) continue
        if (oid !in reachable) {
            return ValidationResult(false, "ORPHANED_STEP", "Step $oid is not reachable from START")
        }
    }
```

- [ ] **Step 4: Run — expect pass.** **Step 5: Commit**
```
git add src/commonMain/kotlin/com/trajectoryruntime/engine/Validator.kt src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchValidatorTest.kt
git commit -m "feat(kmp validator): exempt catch networks from orphan check"
```

### Task KB3: Structural + cross-reference + topology rules

**Files:** Modify `Validator.kt` (new `tryCatchValidation` phase, wired into `validate()`), `TryCatchValidatorTest.kt`.

- [ ] **Step 1: Add failing tests** — append (one per rule; mirrors the TS plan's B3–B5):
```kotlin
  @Test fun `CATCH_WRONG_DEGREE`() {
    val wf = baseWithCatch().replace(
      """{"from_step_id":"c1","to_step_id":"r1"}""",
      """{"from_step_id":"c1","to_step_id":"r1"},{"from_step_id":"s1","to_step_id":"c1"}""")
    assertEquals("CATCH_WRONG_DEGREE", validate(wfMap(wf)).error_code)
  }
  @Test fun `RETURN_WRONG_DEGREE`() {
    val wf = baseWithCatch().replace(
      """{"from_step_id":"c1","to_step_id":"r1"}""",
      """{"from_step_id":"c1","to_step_id":"r1"},{"from_step_id":"r1","to_step_id":"s3"}""")
    assertEquals("RETURN_WRONG_DEGREE", validate(wfMap(wf)).error_code)
  }
  @Test fun `DUPLICATE_TRY_MODE`() {
    assertEquals("DUPLICATE_TRY_MODE",
      validate(wfMap(baseWithCatch(trySpecJson = """[{"mode":"ERROR","catch_id":"C1"},{"mode":"ERROR","catch_id":"C1"}]"""))).error_code)
  }
  @Test fun `MISSING_RESTART_MODE`() {
    assertEquals("MISSING_RESTART_MODE", validate(wfMap(baseWithCatch(returnJson = """{"command":"RESTART"}"""))).error_code)
  }
  @Test fun `MISSING_GOTO_TARGET`() {
    assertEquals("MISSING_GOTO_TARGET", validate(wfMap(baseWithCatch(returnJson = """{"command":"GOTO"}"""))).error_code)
  }
  @Test fun `UNMATCHED_TRY`() {
    assertEquals("UNMATCHED_TRY", validate(wfMap(baseWithCatch(trySpecJson = """[{"mode":"ERROR","catch_id":"Ghost"}]"""))).error_code)
  }
  @Test fun `DUPLICATE_CATCH_ID`() {
    val wf = baseWithCatch(extra = ""","{"local_id":"C2","oid":"c2","version":"1.0.0","last_modified_date":"$DATE","step_type":"CATCH","catch_id":"C1"},{"local_id":"R2","oid":"r2","version":"1.0.0","last_modified_date":"$DATE","step_type":"RETURN","return_config":{"command":"ABANDON"}}""")
      .replace(""""connections":[""", """"connections":[{"from_step_id":"c2","to_step_id":"r2"},""")
    assertEquals("DUPLICATE_CATCH_ID", validate(wfMap(wf)).error_code)
  }
  @Test fun `GOTO_TARGET_NOT_FOUND`() {
    assertEquals("GOTO_TARGET_NOT_FOUND", validate(wfMap(baseWithCatch(returnJson = """{"command":"GOTO","goto_step_oid":"ghost"}"""))).error_code)
  }
  @Test fun `GOTO_TARGET_IN_CATCH`() {
    assertEquals("GOTO_TARGET_IN_CATCH", validate(wfMap(baseWithCatch(returnJson = """{"command":"GOTO","goto_step_oid":"c1"}"""))).error_code)
  }
  @Test fun `CROSS_NETWORK_EDGE`() {
    val wf = baseWithCatch().replace(
      """{"from_step_id":"c1","to_step_id":"r1"}""",
      """{"from_step_id":"c1","to_step_id":"r1"},{"from_step_id":"s2","to_step_id":"r1"}""")
    assertEquals("CROSS_NETWORK_EDGE", validate(wfMap(wf)).error_code)
  }
  @Test fun `accepts an orphaned CATCH (valid true)`() {
    assertEquals(true, validate(wfMap(baseWithCatch(trySpecJson = "[]"))).valid)
  }
```

- [ ] **Step 2: Run — expect failures.**

- [ ] **Step 3: Implement** — in `Validator.kt`, add the phase and wire it into `validate()` right after `semanticValidation(workflow)?.let { return it }`:
```kotlin
    // Phase A1.5: TRY/CATCH/RETURN structural + cross-reference + topology rules
    tryCatchValidation(workflow)?.let { return it }
```
Add the function:
```kotlin
private fun tryCatchValidation(workflow: Map<String, Any?>): ValidationResult? {
    @Suppress("UNCHECKED_CAST")
    val steps = workflow["steps"] as List<Map<String, Any?>>
    @Suppress("UNCHECKED_CAST")
    val connections = workflow["connections"] as List<Map<String, Any?>>
    fun fail(code: String, msg: String) = ValidationResult(false, code, msg)

    val tryScoped = setOf("ACTION PROXY", "WAIT ACTION PROXY")
    val modes = setOf("ERROR", "ABORT", "TIMEOUT")
    val commands = setOf("ABANDON", "RESTART", "GOTO", "RETRY")

    // Structural (per step)
    for (step in steps) {
        val type = normalizeStepType(step["step_type"] as String)
        val oid = step["oid"] as String
        val inDeg = connections.count { it["to_step_id"] == oid }
        val outDeg = connections.count { it["from_step_id"] == oid }
        if (type == "CATCH" && (inDeg != 0 || outDeg != 1))
            return fail("CATCH_WRONG_DEGREE", "CATCH $oid must have 0 in / 1 out (has $inDeg in, $outDeg out)")
        if (type == "RETURN" && (inDeg != 1 || outDeg != 0))
            return fail("RETURN_WRONG_DEGREE", "RETURN $oid must have 1 in / 0 out (has $inDeg in, $outDeg out)")

        @Suppress("UNCHECKED_CAST")
        val tries = step["try_specifications"] as? List<Map<String, Any?>>
        if (!tries.isNullOrEmpty()) {
            if (type !in tryScoped) return fail("TRY_ON_INVALID_STEP", "try_specifications not allowed on $type")
            val seen = HashSet<String>()
            for (t in tries) {
                val m = t["mode"] as? String
                if (m !in modes) return fail("INVALID_TRY_MODE", "invalid try mode '$m'")
                if (!seen.add(m!!)) return fail("DUPLICATE_TRY_MODE", "mode '$m' repeated on $oid")
            }
        }
        if (type == "RETURN") {
            @Suppress("UNCHECKED_CAST")
            val rc = step["return_config"] as? Map<String, Any?>
            if (rc != null) {
                val cmd = rc["command"] as? String
                if (cmd !in commands) return fail("INVALID_RETURN_COMMAND", "invalid RETURN command '$cmd'")
                if (cmd == "RESTART" && rc["restart_mode"] == null) return fail("MISSING_RESTART_MODE", "RESTART needs restart_mode")
                val rm = rc["restart_mode"] as? String
                if (rm != null && rm != "CLEAN" && rm != "KEEP") return fail("INVALID_RESTART_MODE", "invalid restart_mode '$rm'")
                if (cmd == "GOTO" && rc["goto_step_oid"] == null) return fail("MISSING_GOTO_TARGET", "GOTO needs goto_step_oid")
            }
        }
    }

    // Cross-reference
    val catchIds = HashSet<String>()
    for (step in steps) {
        if (normalizeStepType(step["step_type"] as String) != "CATCH") continue
        val cid = step["catch_id"] as? String ?: continue
        if (!catchIds.add(cid)) return fail("DUPLICATE_CATCH_ID", "catch_id '$cid' used by more than one CATCH")
    }
    val partition = partitionCatchNetworks(
        steps.map { Triple(it["oid"] as String, it["step_type"] as String, it["catch_id"] as String?) },
        connections.map { (it["from_step_id"] as String) to (it["to_step_id"] as String) },
    )
    val oidSet = steps.map { it["oid"] as String }.toSet()
    for (step in steps) {
        @Suppress("UNCHECKED_CAST")
        val tries = step["try_specifications"] as? List<Map<String, Any?>>
        for (t in tries ?: emptyList()) {
            val cid = t["catch_id"] as? String
            if (cid !in catchIds) return fail("UNMATCHED_TRY", "TRY references undefined catch_id '$cid'")
        }
        if (normalizeStepType(step["step_type"] as String) == "RETURN") {
            @Suppress("UNCHECKED_CAST")
            val rc = step["return_config"] as? Map<String, Any?>
            val goto = rc?.get("goto_step_oid") as? String
            if (rc?.get("command") == "GOTO" && goto != null) {
                if (goto !in oidSet) return fail("GOTO_TARGET_NOT_FOUND", "GOTO target '$goto' not found")
                if (goto in partition.catchNetworkStepOids) return fail("GOTO_TARGET_IN_CATCH", "GOTO target '$goto' is inside a catch network")
            }
        }
    }

    // Topology
    for (conn in connections) {
        val fromIn = (conn["from_step_id"] as String) in partition.catchNetworkStepOids
        val toIn = (conn["to_step_id"] as String) in partition.catchNetworkStepOids
        if (fromIn != toIn) return fail("CROSS_NETWORK_EDGE", "connection ${conn["from_step_id"]} -> ${conn["to_step_id"]} crosses the catch-network boundary")
    }
    for (step in steps) {
        if (normalizeStepType(step["step_type"] as String) != "CATCH") continue
        val cid = step["catch_id"] as? String ?: continue
        if (cid !in partition.networksByCatchId) return fail("CATCH_WITHOUT_RETURN", "CATCH ${step["oid"]} has no reachable RETURN")
    }
    // ORPHANED_CATCH (catch_id referenced by no TRY) is accepted at runtime (valid:true).
    return null
}
```

- [ ] **Step 4: Run — expect pass.** **Step 5: Commit**
```
git add src/commonMain/kotlin/com/trajectoryruntime/engine/Validator.kt src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchValidatorTest.kt
git commit -m "feat(kmp validator): TRY/CATCH/RETURN structural, cross-ref, topology rules"
```

---

## Phase K-C — Handlers + conformance ACTION PROXY parking

### Task KC1: `activateCatchStep`

**Files:** Modify `StepHandlers.kt`; create `TryCatchHandlersTest.kt`.

- [ ] **Step 1: Write the failing test** — create `src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchHandlersTest.kt`:
```kotlin
package com.trajectoryruntime.engine

import kotlin.test.Test
import kotlin.test.assertEquals

class TryCatchHandlersTest {
  @Test fun `activateCatchStep writes trigger info to declared targets`() {
    val store = PropertyStore()
    store.set("FailureContext.Mode", "")
    store.set("FailureContext.Message", "")
    val step = MasterWorkflowStep(
      local_id = "Catch", oid = "c1", version = "1.0.0", last_modified_date = "d", step_type = "CATCH", catch_id = "C1",
      output_parameter_specifications = listOf(
        OutputParameterSpecification(id = "trigger_reason", target = "FailureContext.Mode"),
        OutputParameterSpecification(id = "error_message", target = "FailureContext.Message"),
      ),
    )
    val ctx = CatchContext("c1", "a1", "Heat Oven", "ERROR", "thermocouple failure", "d")
    activateCatchStep(step, ctx, store)
    assertEquals("ERROR", store.get("FailureContext.Mode"))
    assertEquals("thermocouple failure", store.get("FailureContext.Message"))
  }
}
```

- [ ] **Step 2: Run — expect failure** (unresolved `activateCatchStep`).

- [ ] **Step 3: Implement** — append to `StepHandlers.kt`:
```kotlin
/** Spec §1.2/§3.1: write runtime-supplied trigger info to the CATCH's declared Value Property targets. */
fun activateCatchStep(step: MasterWorkflowStep, ctx: CatchContext, propertyStore: PropertyStore) {
    val fields: Map<String, (CatchContext) -> String> = mapOf(
        "trigger_step" to { c -> c.trigger_step_name },
        "trigger_step_oid" to { c -> c.trigger_step_oid },
        "trigger_reason" to { c -> c.trigger_reason },
        "error_message" to { c -> c.error_message ?: "" },
    )
    for (out in step.output_parameter_specifications ?: emptyList()) {
        val f = fields[out.id] ?: continue           // unknown id: ignore silently (spec §1.2)
        val target = out.target ?: continue
        propertyStore.set(target, f(ctx))
    }
}
```

- [ ] **Step 4: Run — expect pass.** **Step 5: Commit**
```
git add src/commonMain/kotlin/com/trajectoryruntime/engine/StepHandlers.kt src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchHandlersTest.kt
git commit -m "feat(kmp engine): activateCatchStep handler"
```

### Task KC2: ACTION PROXY parks EXECUTING in conformance mode

So the conformance engine (no invoker) can drive an ACTION PROXY to success or failure via `submitAction`, exactly like the TS engine — and the existing `exec-action-proxy-001` fixture (which expects `EXECUTING`) starts passing.

**Files:** Modify `WorkflowEngine.kt` (`activateActionProxy`); create `TryCatchEngineTest.kt`.

- [ ] **Step 1: Write the failing test** — create `src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchEngineTest.kt` with the shared helpers (`wfSpec` from the Test-workflow section) and:
```kotlin
package com.trajectoryruntime.engine

import kotlinx.serialization.json.Json
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertTrue

private val LENIENT = Json { ignoreUnknownKeys = true }
private fun wfSpec(json: String): MasterWorkflowSpecification = LENIENT.decodeFromString(json)
private const val DATE = "2026-05-31T12:00:00.000Z"

/** START→Action(try ERROR→C1)→END + Catch→Return. opts override the TRY and RETURN JSON. */
private fun tryWf(trySpec: String = """[{"mode":"ERROR","catch_id":"C1"}]""",
                  returnJson: String = """{"command":"ABANDON"}""",
                  catchOutputs: String = """[{"id":"trigger_reason","target":"FailureContext.Mode"}]""",
                  extraSteps: String = "", extraConns: String = ""): String = """
{
 "local_id":"wf","oid":"wf-1","version":"1.0.0","last_modified_date":"$DATE","schemaVersion":"4.0",
 "value_property_specifications":[{"name":"FailureContext","entries":[{"name":"Mode","value":""}]}],
 "steps":[
  {"local_id":"Start","oid":"s1","version":"1.0.0","last_modified_date":"$DATE","step_type":"START"},
  {"local_id":"Action","oid":"s2","version":"1.0.0","last_modified_date":"$DATE","step_type":"ACTION PROXY","try_specifications":$trySpec},
  {"local_id":"End","oid":"s3","version":"1.0.0","last_modified_date":"$DATE","step_type":"END"},
  {"local_id":"Catch","oid":"c1","version":"1.0.0","last_modified_date":"$DATE","step_type":"CATCH","catch_id":"C1","output_parameter_specifications":$catchOutputs},
  {"local_id":"Ret","oid":"r1","version":"1.0.0","last_modified_date":"$DATE","step_type":"RETURN","return_config":$returnJson}
  $extraSteps
 ],
 "connections":[{"from_step_id":"s1","to_step_id":"s2"},{"from_step_id":"s2","to_step_id":"s3"},{"from_step_id":"c1","to_step_id":"r1"}$extraConns]
}"""

class TryCatchEngineTest {
  @Test fun `ACTION PROXY parks EXECUTING with no invoker`() {
    val engine = WorkflowEngine(wfSpec(tryWf()))
    engine.start()
    assertTrue(engine.getActiveSteps().any { it.step.oid == "s2" && it.step.step.step_type == "ACTION PROXY" })
    assertEquals(WorkflowState.RUNNING, engine.getWorkflowState())
  }
}
```
(Note `ActiveStepInfo.step` is a `StepInstance`; its `.step` is the `MasterWorkflowStep`. So `it.step.oid` is the StepInstance oid and `it.step.step.step_type` the raw type.)

- [ ] **Step 2: Run — expect failure** (`activateActionProxy` ERRORs → workflow ERRORED, s2 not active).

- [ ] **Step 3: Implement** — in `WorkflowEngine.kt` `activateActionProxy`, replace the no-invoker branch (lines 1002-1008) so it parks EXECUTING instead of erroring:
```kotlin
        val invoker = actionInvoker
        if (invoker == null) {
            // Conformance / headless mode: no Action Container. Park the step like a user-action step
            // so submitAction (success or 'fail') can drive it — mirrors the TS engine.
            recordTrace(target.oid, "EXECUTING")
            target.state = StepState.EXECUTING
            pendingUserSteps.add(target.oid)
            return
        }
```

- [ ] **Step 4: Run — expect pass.** Also re-run the full suite — the existing `exec-action-proxy-001` fixture (expects `step-act:EXECUTING`, `workflow_state:RUNNING`) now passes through `ConformanceRunner` too.

- [ ] **Step 5: Commit**
```
git add src/commonMain/kotlin/com/trajectoryruntime/engine/WorkflowEngine.kt src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchEngineTest.kt
git commit -m "feat(kmp engine): ACTION PROXY parks EXECUTING in conformance mode"
```

---

## Phase K-D — Fail signal + TRY routing

(Mirrors web/TS Plan 1 Phase D, in Kotlin. Uses `TryCatchEngineTest.kt` + the `tryWf(...)` helper from KC2.)

### Task KD1: `fail` signal — uncaught failure errors the workflow

**Files:** Modify `WorkflowEngine.kt`; modify `TryCatchEngineTest.kt`.

- [ ] **Step 1: Add failing test** — append to `TryCatchEngineTest`:
```kotlin
  @Test fun `uncaught failure errors the workflow`() {
    val engine = WorkflowEngine(wfSpec(tryWf(trySpec = "[]"))) // Action has no TRY
    engine.start()
    engine.submitAction(UserAction(step_oid = "s2", action = "fail", failure_mode = "ERROR", error = "boom"), 0)
    assertEquals(WorkflowState.ERRORED, engine.getWorkflowState())
    assertTrue(engine.getTrace().any { it.step_oid == "s2" && it.state == "ERRORED" })
  }
```

- [ ] **Step 2: Run — expect failure** (`submitAction` throws "not EXECUTING" is false — s2 IS EXECUTING after KC2 — but the `"fail"` action falls through to the COMPLETED path, so state is COMPLETED not ERRORED).

- [ ] **Step 3: Implement** — in `WorkflowEngine.kt`:

Add the field (next to the other private fields):
```kotlin
    private val active_catches = mutableMapOf<String, CatchContext>()
```
In `submitAction`, immediately after the `EXECUTING` guard (`if (stepInstance.state != StepState.EXECUTING) throw ...`), add:
```kotlin
        if (action.action == "fail") {
            handleStepFailure(stepInstance, action.failure_mode ?: "ERROR", action.error, actionIndex)
            return
        }
```
Add the method (TRY routing added in KD2; default = uncaught → ERRORED, mirroring the SCRIPT-error path at `activateStepAfterResources`):
```kotlin
    private fun handleStepFailure(stepInstance: StepInstance, mode: String, error: String?, actionIndex: Int) {
        recordTrace(stepInstance.oid, "ERRORED", actionIndex, error)
        stepInstance.state = StepState.ERRORED
        workflowState = WorkflowState.ERRORED
        pendingUserSteps.remove(stepInstance.oid)
        val known = mutableSetOf<String>()
        collectKnownStepOids(known)
        resourceManager?.cancelQueuedWaiters(known)
    }
```

- [ ] **Step 4: Run — expect pass.** **Step 5: Commit**
```
git add src/commonMain/kotlin/com/trajectoryruntime/engine/WorkflowEngine.kt src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchEngineTest.kt
git commit -m "feat(kmp engine): fail signal — uncaught failure errors the workflow"
```

### Task KD2: TRY routing → CATCH activation

**Files:** Modify `WorkflowEngine.kt`, `TryCatchEngineTest.kt`.

- [ ] **Step 1: Add failing test** — append:
```kotlin
  @Test fun `matching failure activates the CATCH network and writes trigger info`() {
    val engine = WorkflowEngine(wfSpec(tryWf()))
    engine.start()
    engine.submitAction(UserAction(step_oid = "s2", action = "fail", failure_mode = "ERROR", error = "thermocouple"), 0)
    val states = engine.getTrace().map { "${it.step_oid}:${it.state}" }
    assertTrue(states.contains("c1:COMPLETED"), "CATCH not activated: $states")
    assertEquals("ERROR", engine.getProperties()["FailureContext.Mode"])
    assertNotEquals(WorkflowState.ERRORED, engine.getWorkflowState())
  }
```
(Add `import kotlin.test.assertNotEquals`.)

- [ ] **Step 2: Run — expect failure.**

- [ ] **Step 3: Implement** — in `WorkflowEngine.kt`:

In `activateStepAfterResources`, inside the `if (isAutoCompleting(target.stepType))` block, before `recordTrace(target.oid, "COMPLETED")`, add the CATCH pre-step:
```kotlin
            if (target.stepType == "CATCH") {
                active_catches[target.oid]?.let { activateCatchStep(target.step, it, propertyStore) }
            }
```
Add the helpers + replace `handleStepFailure` with the match-then-fallback version:
```kotlin
    private fun findCatchByCatchId(catchId: String): StepInstance? =
        steps.values.firstOrNull { it.stepType == "CATCH" && it.step.catch_id == catchId }

    private fun buildPartition(): CatchNetworkPartition = partitionCatchNetworks(
        steps.values.map { Triple(it.oid, it.stepType, it.step.catch_id) },
        connections.map { it.from_step_id to it.to_step_id },
    )

    private fun handleStepFailure(stepInstance: StepInstance, mode: String, error: String?, actionIndex: Int) {
        val matching = stepInstance.step.try_specifications?.firstOrNull { it.mode == mode }
        val catchStep = matching?.let { findCatchByCatchId(it.catch_id) }
        if (matching != null && catchStep != null) {
            if (active_catches.containsKey(catchStep.oid)) {
                // CATCH_REENTRY (spec §6.4)
                recordTrace(stepInstance.oid, "ERRORED", actionIndex, "CATCH_REENTRY on ${matching.catch_id}")
                stepInstance.state = StepState.ERRORED
                workflowState = WorkflowState.ABORTED
                return
            }
            active_catches[catchStep.oid] = CatchContext(
                catch_oid = catchStep.oid,
                trigger_step_oid = stepInstance.oid,
                trigger_step_name = stepInstance.step.local_id,
                trigger_reason = mode,
                error_message = error,
                activated_at = "", // audit-only; conformance is time-free
            )
            recordTrace(stepInstance.oid, "IDLE", actionIndex)
            stepInstance.state = StepState.IDLE
            pendingUserSteps.remove(stepInstance.oid)
            activateStep(catchStep)
            drainCompletionQueue()
            return
        }
        // No matching TRY — uncaught failure errors the workflow.
        recordTrace(stepInstance.oid, "ERRORED", actionIndex, error)
        stepInstance.state = StepState.ERRORED
        workflowState = WorkflowState.ERRORED
        pendingUserSteps.remove(stepInstance.oid)
        val known = mutableSetOf<String>()
        collectKnownStepOids(known)
        resourceManager?.cancelQueuedWaiters(known)
    }
```

- [ ] **Step 4: Run — expect pass** (KD1's no-TRY test still passes via the fallback).

- [ ] **Step 5: Commit**
```
git add src/commonMain/kotlin/com/trajectoryruntime/engine/WorkflowEngine.kt src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchEngineTest.kt
git commit -m "feat(kmp engine): TRY routing activates CATCH network"
```

---

## Phase K-E — RETURN command dispatch

RETURN is intercepted before the generic auto-complete path (so its terminal/redirective effect is final).

### Task KE1: RETURN scaffold + ABANDON

**Files:** Modify `WorkflowEngine.kt`, `TryCatchEngineTest.kt`.

- [ ] **Step 1: Add failing test** — append:
```kotlin
  @Test fun `RETURN ABANDON aborts the workflow`() {
    val engine = WorkflowEngine(wfSpec(tryWf(returnJson = """{"command":"ABANDON"}""")))
    engine.start()
    engine.submitAction(UserAction(step_oid = "s2", action = "fail", failure_mode = "ERROR", error = "x"), 0)
    assertEquals(WorkflowState.ABORTED, engine.getWorkflowState())
  }
```

- [ ] **Step 2: Run — expect failure** (RETURN no-ops; workflow not ABORTED).

- [ ] **Step 3: Implement** — in `WorkflowEngine.kt`:

In `activateStepAfterResources`, add a RETURN intercept **before** the `if (isAutoCompleting(...))` block (right after the `WORKFLOW PROXY` block at ~line 430):
```kotlin
        if (target.stepType == "RETURN") {
            recordTrace(target.oid, "COMPLETED")
            target.state = StepState.COMPLETED
            dispatchReturn(target)
            return // RETURN's command is terminal/redirective — do not fall through.
        }
```
Add the dispatcher + ABANDON + cleanup (stub `returnRestart`/`returnGoto`/`returnRetry` as empty for now; filled in KE2–KE4):
```kotlin
    private fun dispatchReturn(returnStep: StepInstance) {
        val rc = returnStep.step.return_config ?: return
        val catchOid = findActiveCatchForReturn(returnStep.oid)
        val ctx = catchOid?.let { active_catches[it] }
        when (rc.command) {
            "ABANDON" -> returnAbandon()
            "RESTART" -> returnRestart(rc.restart_mode ?: "KEEP")
            "GOTO" -> rc.goto_step_oid?.let { returnGoto(it) }
            "RETRY" -> returnRetry(ctx)
        }
        if (catchOid != null) {
            cleanupCatchNetwork(catchOid)
            active_catches.remove(catchOid)
        }
    }
    private fun findActiveCatchForReturn(returnOid: String): String? {
        for ((catchId, net) in buildPartition().networksByCatchId) {
            if (returnOid in net) {
                val cs = findCatchByCatchId(catchId)
                if (cs != null && active_catches.containsKey(cs.oid)) return cs.oid
            }
        }
        return null
    }
    private fun cleanupCatchNetwork(catchOid: String) {
        val cid = steps[catchOid]?.step?.catch_id ?: return
        val net = buildPartition().networksByCatchId[cid] ?: return
        for (oid in net) {
            val st = steps[oid] ?: continue
            if (st.state != StepState.IDLE) { recordTrace(oid, "IDLE"); st.state = StepState.IDLE }
        }
    }
    private fun returnAbandon() {
        for (step in steps.values) {
            if (step.state in ACTIVE_STEP_STATES) { recordTrace(step.oid, "IDLE"); step.state = StepState.IDLE }
        }
        completionQueue.clear()
        releaseAllResources()
        workflowState = WorkflowState.ABORTED
    }
    private fun returnRestart(mode: String) { /* KE2 */ }
    private fun returnGoto(gotoOid: String) { /* KE3 */ }
    private fun returnRetry(ctx: CatchContext?) { /* KE4 */ }
    private fun resetStepInline(oid: String) {
        val step = steps[oid] ?: return
        if (step.state != StepState.IDLE) recordTrace(oid, "IDLE")
        step.state = StepState.IDLE
        routingContext.remove(oid); pendingResources.remove(oid); pendingUserSteps.remove(oid)
        stepParameterSnapshots.remove(oid); waitAllTracking.remove(oid); activeChildEngines.remove(oid)
    }
```

- [ ] **Step 4: Run — expect pass.** **Step 5: Commit**
```
git add src/commonMain/kotlin/com/trajectoryruntime/engine/WorkflowEngine.kt src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchEngineTest.kt
git commit -m "feat(kmp engine): RETURN dispatch scaffold + ABANDON"
```

### Task KE2: RESTART (CLEAN and KEEP)

- [ ] **Step 1: Add failing tests** — append:
```kotlin
  @Test fun `RESTART KEEP re-runs from START and preserves properties`() {
    val engine = WorkflowEngine(wfSpec(tryWf(returnJson = """{"command":"RESTART","restart_mode":"KEEP"}""")))
    engine.start()
    engine.submitAction(UserAction(step_oid = "s2", action = "fail", failure_mode = "ERROR", error = "x"), 0)
    assertEquals(WorkflowState.RUNNING, engine.getWorkflowState())
    assertEquals("ERROR", engine.getProperties()["FailureContext.Mode"])
    assertTrue(engine.getActiveSteps().any { it.step.oid == "s2" })
  }
  @Test fun `RESTART CLEAN resets properties to defaults`() {
    val engine = WorkflowEngine(wfSpec(tryWf(returnJson = """{"command":"RESTART","restart_mode":"CLEAN"}""")))
    engine.start()
    engine.submitAction(UserAction(step_oid = "s2", action = "fail", failure_mode = "ERROR", error = "x"), 0)
    assertEquals("", engine.getProperties()["FailureContext.Mode"])
  }
```

- [ ] **Step 2: Run — expect failure.**

- [ ] **Step 3: Implement** — replace the `returnRestart` stub:
```kotlin
    private fun returnRestart(mode: String) {
        for (step in steps.values) {
            if (step.state != StepState.IDLE) { recordTrace(step.oid, "IDLE"); step.state = StepState.IDLE }
        }
        completionQueue.clear(); pendingUserSteps.clear(); active_catches.clear()
        releaseAllResources()
        if (mode == "CLEAN") propertyStore.initializeFromWorkflow(workflow) // same call the ctor uses
        val startStep = steps.values.firstOrNull { it.stepType == "START" }
        if (startStep != null) {
            recordTrace(startStep.oid, "COMPLETED")
            startStep.state = StepState.COMPLETED
            completionQueue.addLast(startStep.oid)
        }
    }
```
(The outer `drainCompletionQueue` that invoked this RETURN continues and drains the re-pushed START. The `cleanupCatchNetwork`/`active_catches.remove` in `dispatchReturn` after this are harmless no-ops since RESTART already cleared everything.)

- [ ] **Step 4: Run — expect pass.** **Step 5: Commit**
```
git add src/commonMain/kotlin/com/trajectoryruntime/engine/WorkflowEngine.kt src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchEngineTest.kt
git commit -m "feat(kmp engine): RETURN RESTART (CLEAN re-inits properties; KEEP preserves)"
```

### Task KE3: GOTO

- [ ] **Step 1: Add failing test** — append:
```kotlin
  @Test fun `RETURN GOTO resumes the main flow at the target`() {
    val wf = tryWf(
      returnJson = """{"command":"GOTO","goto_step_oid":"m1"}""",
      extraSteps = ""","{"local_id":"Mid","oid":"m1","version":"1.0.0","last_modified_date":"$DATE","step_type":"USER_INTERACTION"}""")
      .replace(
        """"connections":[{"from_step_id":"s1","to_step_id":"s2"},{"from_step_id":"s2","to_step_id":"s3"}""",
        """"connections":[{"from_step_id":"s1","to_step_id":"s2"},{"from_step_id":"s2","to_step_id":"m1"},{"from_step_id":"m1","to_step_id":"s3"}""")
    val engine = WorkflowEngine(wfSpec(wf))
    engine.start()
    engine.submitAction(UserAction(step_oid = "s2", action = "fail", failure_mode = "ERROR", error = "x"), 0)
    assertTrue(engine.getActiveSteps().any { it.step.oid == "m1" }, "GOTO target not active")
    assertNotEquals(WorkflowState.ERRORED, engine.getWorkflowState())
  }
```

- [ ] **Step 2: Run — expect failure.**

- [ ] **Step 3: Implement** — replace the `returnGoto` stub:
```kotlin
    private fun returnGoto(gotoOid: String) {
        val target = steps[gotoOid] ?: return
        if (target.state != StepState.IDLE) resetStepInline(gotoOid)
        activateStep(target) // branch-local
    }
```

- [ ] **Step 4: Run — expect pass.** **Step 5: Commit**
```
git add src/commonMain/kotlin/com/trajectoryruntime/engine/WorkflowEngine.kt src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchEngineTest.kt
git commit -m "feat(kmp engine): RETURN GOTO resumes main flow at target"
```

### Task KE4: RETRY

- [ ] **Step 1: Add failing test** — append:
```kotlin
  @Test fun `RETURN RETRY re-invokes the trigger; second attempt succeeds`() {
    val engine = WorkflowEngine(wfSpec(tryWf(returnJson = """{"command":"RETRY"}""")))
    engine.start()
    engine.submitAction(UserAction(step_oid = "s2", action = "fail", failure_mode = "ERROR", error = "x"), 0)
    assertTrue(engine.getActiveSteps().any { it.step.oid == "s2" }, "s2 not re-activated")
    engine.submitAction(UserAction(step_oid = "s2", action = "submit"), 1)
    assertEquals(WorkflowState.COMPLETED, engine.getWorkflowState())
  }
```

- [ ] **Step 2: Run — expect failure.**

- [ ] **Step 3: Implement** — replace the `returnRetry` stub:
```kotlin
    private fun returnRetry(ctx: CatchContext?) {
        if (ctx == null) return
        val trigger = steps[ctx.trigger_step_oid] ?: return
        if (trigger.state != StepState.IDLE) resetStepInline(ctx.trigger_step_oid)
        activateStep(trigger) // ACTION PROXY → EXECUTING again
    }
```

- [ ] **Step 4: Run — expect pass.** **Step 5: Commit**
```
git add src/commonMain/kotlin/com/trajectoryruntime/engine/WorkflowEngine.kt src/jvmTest/kotlin/com/trajectoryruntime/engine/TryCatchEngineTest.kt
git commit -m "feat(kmp engine): RETURN RETRY re-invokes the trigger action"
```

---

## Phase K-F — Shared conformance capstone

These JSON fixtures live in `spec/conformance/` and are auto-discovered by **both** the TS runner (`engines/web/npm run conformance`) and the Kotlin `ConformanceRunner` (`:jvmTest`) — one fixture proves both engines. They assert the **deterministic** `expected.valid`/`error_code` (validation) and `expected.workflow_state` + `final_properties` (execution); `execution_trace` is intentionally omitted (the runner only compares it when present), so we avoid brittle cross-engine trace-order coupling.

> **Prerequisite:** the web/TS plan must be complete, so these pass in `engines/web` too. After this phase, run BOTH suites.

### Task KF1: Allow `ERRORED` as a fixture workflow_state

**Files:** Modify `spec/conformance/test-fixture-schema.json`.

- [ ] **Step 1:** Add `"ERRORED"` to the `expected.workflow_state` enum (lines 114-118):
```json
        "workflow_state": {
          "type": "string",
          "enum": ["IDLE", "RUNNING", "COMPLETED", "ABORTED", "STOPPED", "ERRORED"],
          "description": "Expected final workflow instance state."
        }
```
- [ ] **Step 2: Commit**
```
git add spec/conformance/test-fixture-schema.json
git commit -m "chore(conformance): allow ERRORED workflow_state in fixtures"
```

### Task KF2: Validation fixtures V1–V7

**Files:** Create seven files in `spec/conformance/validation/`. Each is the **same workflow as the correspondingly-named `TryCatchValidatorTest` case**, saved as a fixture with the listed `expected`. (The workflows are fully specified by those tests; serialize them verbatim into JSON files.)

Exemplar — `spec/conformance/validation/val-try-catch-003-duplicate-catch-id.json` (full, copy this shape for the others):
```json
{
  "test_id": "val-try-catch-003",
  "name": "Duplicate catch_id is rejected",
  "category": "validation",
  "tags": ["try-catch", "validation"],
  "workflow": {
    "local_id": "wf", "oid": "wf-1", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "schemaVersion": "4.0",
    "steps": [
      { "local_id": "Start", "oid": "s1", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "step_type": "START" },
      { "local_id": "Action", "oid": "s2", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "step_type": "ACTION PROXY", "try_specifications": [{ "mode": "ERROR", "catch_id": "C1" }] },
      { "local_id": "End", "oid": "s3", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "step_type": "END" },
      { "local_id": "Catch", "oid": "c1", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "step_type": "CATCH", "catch_id": "C1" },
      { "local_id": "Ret", "oid": "r1", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "step_type": "RETURN", "return_config": { "command": "ABANDON" } },
      { "local_id": "Catch2", "oid": "c2", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "step_type": "CATCH", "catch_id": "C1" },
      { "local_id": "Ret2", "oid": "r2", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "step_type": "RETURN", "return_config": { "command": "ABANDON" } }
    ],
    "connections": [
      { "from_step_id": "s1", "to_step_id": "s2" }, { "from_step_id": "s2", "to_step_id": "s3" },
      { "from_step_id": "c1", "to_step_id": "r1" }, { "from_step_id": "c2", "to_step_id": "r2" }
    ]
  },
  "expected": { "valid": false, "error_code": "DUPLICATE_CATCH_ID" }
}
```
The full set (each `expected` block + the source test that defines the workflow):
| File | Source test | `expected` |
|---|---|---|
| `val-try-catch-001-cross-network-edge.json` | `CROSS_NETWORK_EDGE` | `{valid:false, error_code:"CROSS_NETWORK_EDGE"}` |
| `val-try-catch-002-catch-without-return.json` | (CATCH→non-RETURN dead end) | `{valid:false, error_code:"CATCH_WITHOUT_RETURN"}` |
| `val-try-catch-003-duplicate-catch-id.json` | `DUPLICATE_CATCH_ID` | `{valid:false, error_code:"DUPLICATE_CATCH_ID"}` |
| `val-try-catch-004-unmatched-try.json` | `UNMATCHED_TRY` | `{valid:false, error_code:"UNMATCHED_TRY"}` |
| `val-try-catch-005-goto-into-catch.json` | `GOTO_TARGET_IN_CATCH` | `{valid:false, error_code:"GOTO_TARGET_IN_CATCH"}` |
| `val-try-catch-006-try-on-script.json` | TRY on a `SCRIPT` step | `{valid:false, error_code:"TRY_ON_INVALID_STEP"}` |
| `val-try-catch-007-orphaned-catch.json` | `accepts an orphaned CATCH` | `{valid:true}` |

- [ ] **Step 1:** Create all seven files.
- [ ] **Step 2: Run BOTH validators** — `cd engines/web && npm run conformance` AND `cd engines/kmp-engine && .\gradlew.bat :jvmTest`. Both must report these 7 fixtures passing (identical `error_code` from both validators — this is the cross-engine validator parity proof).
- [ ] **Step 3: Commit**
```
git add spec/conformance/validation/val-try-catch-*.json
git commit -m "test(conformance): TRY/CATCH/RETURN validation fixtures V1-V7 (both engines)"
```

### Task KF3: Execution fixtures

**Files:** Create files in `spec/conformance/execution/`. Each drives a failure via a `user_actions` entry with `action:"fail"` and asserts `workflow_state` (+ `final_properties` where deterministic).

Exemplar — `spec/conformance/execution/exec-try-catch-001-error-to-abandon.json` (full):
```json
{
  "test_id": "exec-try-catch-001",
  "name": "ERROR routes to CATCH; RETURN ABANDON aborts the workflow",
  "category": "execution",
  "tags": ["try-catch", "abandon"],
  "workflow": {
    "local_id": "wf", "oid": "wf-1", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "schemaVersion": "4.0",
    "value_property_specifications": [{ "name": "FailureContext", "entries": [{ "name": "Mode", "value": "" }] }],
    "steps": [
      { "local_id": "Start", "oid": "s1", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "step_type": "START" },
      { "local_id": "Action", "oid": "s2", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "step_type": "ACTION PROXY", "try_specifications": [{ "mode": "ERROR", "catch_id": "C1" }] },
      { "local_id": "End", "oid": "s3", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "step_type": "END" },
      { "local_id": "Catch", "oid": "c1", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "step_type": "CATCH", "catch_id": "C1", "output_parameter_specifications": [{ "id": "trigger_reason", "target": "FailureContext.Mode" }] },
      { "local_id": "Ret", "oid": "r1", "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "step_type": "RETURN", "return_config": { "command": "ABANDON" } }
    ],
    "connections": [
      { "from_step_id": "s1", "to_step_id": "s2" }, { "from_step_id": "s2", "to_step_id": "s3" }, { "from_step_id": "c1", "to_step_id": "r1" }
    ]
  },
  "user_actions": [{ "step_oid": "s2", "action": "fail", "failure_mode": "ERROR", "error": "thermocouple failure" }],
  "expected": { "valid": true, "workflow_state": "ABORTED", "final_properties": { "FailureContext.Mode": "ERROR" } }
}
```
The full set (each = the same workflow as the named `TryCatchEngineTest` case, with the listed `user_actions` + `expected`):
| File | Drives | `expected` |
|---|---|---|
| `exec-try-catch-001-error-to-abandon.json` | `fail ERROR` → ABANDON | `{workflow_state:"ABORTED", final_properties:{"FailureContext.Mode":"ERROR"}}` |
| `exec-try-catch-002-timeout-to-retry.json` | `fail TIMEOUT` then `submit` → RETRY succeeds | `{workflow_state:"COMPLETED"}` |
| `exec-try-catch-003-abort-to-goto.json` | `fail ABORT` → GOTO Mid, then `submit` Mid | `{workflow_state:"COMPLETED"}` |
| `exec-try-catch-004-restart-keep.json` | `fail ERROR` → RESTART KEEP | `{workflow_state:"RUNNING", final_properties:{"FailureContext.Mode":"ERROR"}}` |
| `exec-try-catch-005-restart-clean.json` | `fail ERROR` → RESTART CLEAN | `{workflow_state:"RUNNING", final_properties:{"FailureContext.Mode":""}}` |
| `exec-try-catch-006-uncaught-errors.json` | `fail ERROR` with no TRY | `{workflow_state:"ERRORED"}` |

- [ ] **Step 1:** Create the files (002/003 need a second `user_actions` `submit`; 003 needs a `Mid` USER_INTERACTION on the main flow as the GOTO target — copy the GOTO test's workflow).
- [ ] **Step 2: Run BOTH engines** — `cd engines/web && npm run conformance` AND `cd engines/kmp-engine && .\gradlew.bat :jvmTest`. Every fixture must pass in **both** (identical `workflow_state` + `final_properties`). If a `final_properties` value differs between engines, that's a real parity bug to fix — not a fixture tweak.
- [ ] **Step 3: Commit**
```
git add spec/conformance/execution/exec-try-catch-*.json
git commit -m "test(conformance): TRY/CATCH/RETURN execution fixtures (cross-engine parity)"
```

---

## Self-Review (run after completing all tasks)

- [ ] **Kotlin suite green:** `cd engines/kmp-engine && .\gradlew.bat :jvmTest` — all `TryCatch*Test` + the full `ConformanceRunner` (no regressions; `exec-action-proxy-001` now passes).
- [ ] **Cross-engine parity proven:** the `exec-try-catch-*`/`val-try-catch-*` fixtures pass in BOTH `npm run conformance` (TS) and `:jvmTest` (Kotlin). This is the capstone deliverable.
- [ ] **No placeholders:** confirm the `returnRestart`/`returnGoto`/`returnRetry` stubs from KE1 were all replaced.
- [ ] **Validator parity:** every error code added in TS Plan 1 Phase B has a matching Kotlin rule in `tryCatchValidation` (CATCH/RETURN degree, TRY scope/mode, RETURN command fields, DUPLICATE_CATCH_ID, UNMATCHED_TRY, GOTO targets, CROSS_NETWORK_EDGE, CATCH_WITHOUT_RETURN). 
- [ ] **Deviations noted:** `CatchContext.activated_at` is `""` (audit-only; commonMain has no portable wall clock — wire a real timestamp via expect/actual only if a consumer needs it). `RETURN_WITHOUT_CATCH`/`CATCH_NETWORK_NOT_CONNECTED` covered transitively (same as TS). `release_on_catch` parsed but not acted on (descoped). The android-app coordinator wiring (real failure → fail signal) is a separate follow-up.

## Execution Handoff

This completes the **web + native parity** scope. After both runtime plans:
- The TRY/CATCH/RETURN feature works and is conformance-proven in both engines; the silent-swallow bug is fixed in both.
- **Remaining for full delivery (separate efforts):** (a) the **editor** plan (`2026-05-31-try-catch-editor.md`) — unvalidated against `TrajectoryEditor`; run the same grounding check before executing. (b) the **docs** plan. (c) the **android-app coordinator** wiring (feed real action terminals into the Kotlin engine's `fail` signal + timeout, mirroring web/TS Phase F).

