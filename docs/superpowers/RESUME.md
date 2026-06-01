# Resume — TRY/CATCH/RETURN (paused 2026-05-31)

> **UPDATE 2026-06-01 (later) — 🚧 PLAN 2 (Kotlin native-kmp parity) IN PROGRESS — paused mid-execution:**
> Executing `plans/2026-05-31-try-catch-runtime-native-kmp.md` subagent-driven (per-PHASE implementers + phase-boundary review + the shared conformance capstone as the cross-engine parity gate). Work in `engines/kmp-engine`. Pushed to `origin/main` through `abed166`. Tasks tracked #20–#26.
> - **DONE & pushed:** K-A types+allow-list (`119e811`); K-B validator — partition `d6c6408`, orphan exemption `f90b3db`, tryCatchValidation rules `abed166`. All `TryCatchValidatorTest` pass; full `.\gradlew.bat :jvmTest` = **89/90**, the 1 fail (`exec-action-proxy-001`) is pre-existing and is **fixed by K-C task KC2**.
> - **K-B was NOT phase-reviewed** (review dispatch was interrupted by the user). Code is tested-green + corrections applied — a resuming session may review `git diff 119e811 abed166` or just continue (low risk; the conformance capstone proves parity).
> - **REMAINING:** K-C (KC1 activateCatchStep, KC2 ACTION PROXY parks EXECUTING — plan ~L490-621), K-D (KD1 fail signal, KD2 TRY routing — ~L623-759), K-E (KE1 ABANDON, KE2 RESTART, KE3 GOTO, KE4 RETRY — ~L761-967), K-F (KF1 schema enum, KF2 val fixtures, KF3 exec fixtures — ~L969-1089), Final (full :jvmTest + BOTH conformance suites + whole-Plan-2 review).
>
> **CRITICAL corrections that DEVIATE from the kmp plan — a resuming session MUST apply these (grounded vs real Kotlin code; same fixes already in TS Plan 1):**
> 1. **action_proxy_config:** Kotlin `actionProxyValidation` (Validator.kt ~L413) hard-fails a bare ACTION PROXY (`INVALID_VALIDATION`). `baseWithCatch` (K-A, DONE) already carries `action_proxy_config{action_oid:"act-1",environment_oid:"env-1"}` + `environment_specifications:[{oid:"env-1",included_actions:[{action_oid:"act-1",action_name:"DoThing",action_library:"lib-1"}]}]`. **K-F exec-try-catch fixtures + the orphaned-CATCH validation fixture MUST also include them** (ConformanceRunner validates before executing, in BOTH engines). Engine unit tests (tryWf) need NOT (engine doesn't validate).
> 2. **Use `activeCatches` (camelCase)**, not the plan's `active_catches`. Apply in K-D/K-E.
> 3. **`CATCH_WITHOUT_RETURN` is OMITTED** from tryCatchValidation (done in K-B) — editor-time; runtime rejects RETURN-less catch transitively as `ORPHANED_STEP`. So **K-F `val-try-catch-002` must expect `ORPHANED_STEP`**, not CATCH_WITHOUT_RETURN (matches both engines).
> 4. **CROSS_NETWORK_EDGE test** routes the cross edge via an interior node (done K-B) — direct `s2→r1` trips RETURN_WRONG_DEGREE first.
> 5. **returnRestart (KE2)** must idle every step via `resetStepInline(oid)` (clears routingContext/pendingResources/pendingUserSteps/stepParameterSnapshots/waitAllTracking/activeChildEngines), NOT a plain `state=IDLE` loop — matches TS E2 fix.
> 6. **Plan `extra`-param JSON bug:** plan writes `extra = ""","{"local_id"...}"""` → invalid JSON (spurious leading `"`). Use `,{"local_id"...}`. K-B fixed its uses; watch K-E/K-F.
> 7. **KF1 (add "ERRORED" to `spec/conformance/test-fixture-schema.json` workflow_state enum) before KF3** (exec-try-catch-006 asserts ERRORED).
> 8. **K-F is the SHARED parity gate:** after creating fixtures run BOTH `cd engines/web && npm run conformance` (TS) AND `cd engines/kmp-engine && .\gradlew.bat :jvmTest` (Kotlin); each new fixture passes in BOTH. Pre-existing TS-conformance fails to IGNORE: `val-struct-002`, `exec-child-001`, `exec-child-003`. A differing `final_properties` between engines = real parity bug → fix engine, not fixture.
>
> **Verified Kotlin anchors (skip re-grounding):** Validator.kt — `validate()` order pre→semantic→resource→structural→actionProxy; `VALID_STEP_TYPES` @177 (now incl CATCH/RETURN); `normalizeStepType` @5; orphan loop (reachable BFS) in semanticValidation. StepHandlers.kt — `isAutoCompleting` @14 (now incl CATCH/RETURN), `canonicalStepType` @9. Types.kt — `UserAction` @342 (now +failure_mode/error), `MasterWorkflowStep` @148 (now +try_specifications/catch_id/return_config), `StepState` has ABORTED, `WorkflowState{IDLE,RUNNING,COMPLETED,ABORTED,STOPPED,ERRORED}`, `StepInstance{oid,stepType,state,step}`, `ValidationResult(valid,error_code?,error_message?)`; FailureMode/TrySpecification/ReturnConfig/CatchContext added. WorkflowEngine.kt — camelCase private fields (steps:Map, connections:List<WorkflowConnection>, propertyStore, pendingUserSteps, completionQueue:ArrayDeque, workflowState, resourceManager?); `submitAction` EXECUTING-guard then hard-codes COMPLETED (~L194-226 — KD1 adds the `"fail"` branch right after the guard); `activateStepAfterResources` (~L408): ACTION PROXY→activateActionProxy, WORKFLOW PROXY branch, then `isAutoCompleting` block (recordTrace COMPLETED ~L462) — KD2 adds the CATCH pre-step here, KE1 adds the RETURN intercept before the isAutoCompleting block; `activateActionProxy` no-invoker branch ERRORs @~L1001-1008 (KC2 replaces → park EXECUTING + pendingUserSteps.add); `recordTrace(stepOid,state,afterAction?,error?)` @353; `drainCompletionQueue` @365; `collectKnownStepOids` @669; `releaseAllResources` @281, `abortWorkflow` @984, `restartToSteps` @1183 (NO resetStep — KE1 adds `resetStepInline`); getActiveSteps/getTrace/getProperties/getWorkflowState exist. PropertyStore — initializeFromWorkflow/get/set. ConformanceRunner.runExecutionFixture — validate→deserialize→WorkflowEngine(NO invoker)→start→replay user_actions→compare trace+workflow_state(.name)+final_properties; compareTrace ignores `error`; `:jvmTest` runs ALL fixtures. Baseline Kotlin was 73/74 (only exec-action-proxy-001 failing). gradle ~8-30s.
>
> **Resume command:** fresh session in `C:\Trajectory` → "Resume Plan 2 (Kotlin TRY/CATCH parity) — see RESUME.md" → continue from K-C with the plan + the 8 corrections, subagent-driven.

> **UPDATE 2026-06-01 — ✅ WEB/TS RUNTIME PLAN EXECUTED & SHIPPED (Plan 1 complete):**
> - All 18 tasks of `plans/2026-05-31-try-catch-runtime-web-ts.md` (Phases A–F) implemented subagent-driven on `TrajectoryRuntime` `main`, each task spec+code-quality reviewed, and pushed to `origin/main` (`c054102..eee35b0`).
> - Tests: `engines/web` **148/148** (node:test); `engines/web-ui` **53/53** (tsc -b + tsx); conformance **64/67** — the 3 fails (`val-struct-002`, `exec-child-001`, `exec-child-003`) are **PRE-EXISTING since `c054102`** in form-binding + child-workflow areas untouched by this feature (`spec/conformance` has zero diff since baseline).
> - Silent-swallow bug fixed end-to-end: a failed/aborted/timed-out ACTION PROXY now routes to its CATCH network (or ERRORs the workflow if uncaught), driven in production by `WorkflowCoordinator` feeding `submitAction({action:'fail', failure_mode, error})` (the empty-submit Phase-1 stub is gone).
> - Grounded deviations (all sound per final whole-branch review = SHIP): `return-dispatcher.ts` folded into `engine.ts`; `CATCH_WITHOUT_RETURN`/`RETURN_WITHOUT_CATCH`/`CATCH_NETWORK_NOT_CONNECTED` are **editor-time** codes (runtime rejects RETURN-less/disconnected catch islands transitively as `ORPHANED_STEP`, per spec §6.5/§6.6); `release_on_catch` is parsed/validated but **inert** (descoped — no resource holder tracking).
> - Also fixed at the start: 2 pre-existing `form-input-binding-validator` unit-test fixtures (missing schema-required `canvasWidth`/`canvasHeight`/element positions) — `f1ed506`.
> - **REMAINING:** **Plan 2** = `plans/2026-05-31-try-catch-runtime-native-kmp.md` (Kotlin `kmp-engine` parity + the shared `spec/conformance` exec-try-catch fixtures as the cross-engine capstone — NOT STARTED; note `engines/kmp-engine` exists). Then the **editor plan** (still unvalidated vs the real TrajectoryEditor repo — ground it first) and the **docs plan**. Follow-up: resource holder-tracking + real `release_on_catch` (also closes the RETRY × held-exclusive-resource deadlock from spec §5.5).
>
> **— historical re-grounding note (2026-05-31) below —**

> **UPDATE 2026-05-31 (later same day) — re-grounded, do not use the old runtime plan:**
> - The spec + plans were **pushed to `origin/main`** (`ab70157..f57155a`).
> - On resuming, the runtime plan (`plans/2026-05-31-try-catch-runtime.md`) was found **misgrounded** against the real code (assumed Vitest, a single engine with an `onActionInstanceTerminal` consumer, and an error-array validator — none exist). It is now **SUPERSEDED**.
> - Deep recon established the real architecture (see **`plans/2026-05-31-try-catch-IMPLEMENTATION-NOTES.md`** and memory `project-trajectoryruntime-two-engines`): TWO engines (TS `engines/web/src` = web product + reference; Kotlin `kmp-engine` = native), parity via shared `spec/conformance`; action failures are silently swallowed today (no per-step fail signal); validators duplicated (TS + Kotlin); tests are `node:test` (TS) / `.\gradlew.bat :jvmTest` (Kotlin).
> - **Decisions:** scope = **web + native (Kotlin) parity**; `release_on_catch` **descoped** to explicit Release commands; execution mode = **subagent-driven**.
> - **New grounded plans replace the old runtime plan:**
>   - ✅ `plans/2026-05-31-try-catch-runtime-web-ts.md` — authored & code-complete (Phases A–F), ready to execute.
>   - ✅ `plans/2026-05-31-try-catch-runtime-native-kmp.md` — authored & code-complete (Phases K-A…K-F): Kotlin engine parity + the shared conformance fixtures (capstone). Execute AFTER the web/TS plan.
> - **Both runtime plans authored 2026-05-31; neither executed yet.** Remaining beyond runtime: editor plan (unvalidated), docs plan, and android-app coordinator wiring (feed real failures into the Kotlin fail signal).
> - The design spec (`specs/2026-05-31-try-catch-return-design.md`) remains canonical for behaviour; the editor & docs plans are still unvalidated against their repos.

**Status when paused:** Spec + plans committed locally on `main`. Local is 2 commits ahead of `origin/main`. Implementation has NOT started.

## What's done

| Artifact | Path | Commit |
|---|---|---|
| Approved design spec (7 sections + 3 appendices) | `Trajectory/docs/superpowers/specs/2026-05-31-try-catch-return-design.md` | `5dd1368` |
| Runtime implementation plan (30 tasks, 8 phases) | `Trajectory/docs/superpowers/plans/2026-05-31-try-catch-runtime.md` | `f522acf` |
| Editor implementation plan (~20 tasks, 6 phases) | `Trajectory/docs/superpowers/plans/2026-05-31-try-catch-editor.md` | `f522acf` |
| Docs update plan (3 tasks) | `Trajectory/docs/superpowers/plans/2026-05-31-try-catch-docs.md` | `f522acf` |
| `.gitignore` (ignores `/.superpowers/` runtime files) | `Trajectory/.gitignore` | `5dd1368` |

## What's NOT done

- Spec + plans have NOT been pushed to `origin`. Two unpushed commits on `main` (`5dd1368` and `f522acf`).
- No implementation work has begun in any of the three repos.
- Execution mode was not chosen — the question was open at pause: **subagent-driven** (recommended) vs **inline executing-plans**.

## How to resume

Open a fresh Claude Code session in `C:\Trajectory`. Say something like:

> Resuming TRY/CATCH/RETURN work — see `Trajectory/docs/superpowers/RESUME.md`.

Then pick ONE of the following:

### Option 1: Push the work first (zero-friction backup)

```
cd C:/Trajectory/Trajectory
git push origin main
```

Then proceed to Option 2 or 3.

### Option 2: Begin execution — subagent-driven (recommended)

Tell Claude:
> Execute `docs/superpowers/plans/2026-05-31-try-catch-runtime.md` using superpowers:subagent-driven-development.

This dispatches one subagent per task with review checkpoints between tasks. Start with the runtime plan (editor plan depends on its schema work being final). Docs plan is independent and can be parallel or last.

### Option 3: Begin execution — inline

Tell Claude:
> Execute `docs/superpowers/plans/2026-05-31-try-catch-runtime.md` using superpowers:executing-plans.

This walks the plan in the active session with batch checkpoints. Faster turn-cadence; consumes session context fast.

### Option 4: Revise the design before executing

If anything in the spec needs adjustment, the spec file is the canonical source — edit it first, then update the affected plan tasks. The plans cross-reference spec sections; both will need to stay in sync.

## Decisions captured during brainstorming (audit trail)

| Decision | Outcome |
|---|---|
| Symbol shapes | CATCH = wide-top trapezoid; RETURN = narrow-top trapezoid; both at START/END footprint size |
| TRY scope (v1) | `ACTION PROXY` and `WAIT ACTION PROXY` only; generic schema for future extension |
| Catch-network containment | Strict island; no edges between main-flow and catch-network |
| TRY shape | Array of `{mode, catch_id, release_on_catch}` — one entry per mode per step |
| RETRY semantics | Immediate re-invoke; no built-in counter |
| RESTART semantics | Required `restart_mode` ∈ `{CLEAN, KEEP}` |
| Resource cleanup | Per-TRY `release_on_catch` flag (default `true`) |
| Trigger info | CATCH carries `output_parameter_specifications` |
| GOTO scope | Any main-flow step; never a catch-network node |
| Nested TRY | Allowed |
| Branch-local vs global | RETRY/GOTO are branch-local; RESTART/ABANDON are workflow-global |
| Failure classification | Runtime-side; no AC protocol change |
| Schema version | Stays at 4.0 (pure additive) |

Full reasoning lives in the spec file.

## Brainstorm artefacts (transient)

The visual-companion server was running at `http://localhost:53222` (session dir `C:\Trajectory\Trajectory\.superpowers\brainstorm\22624-1780194383\`). It has been gracefully stopped at pause time and is gitignored. The clicked mockups (`symbol-orientation.html`, `catch-network-topology.html`, `try-editor-panel.html`) survive in that session dir if you want to revisit them locally; they will not be committed.

## Open questions / known deferrals

These appeared in the spec's "deferred to implementation" appendix and are not blocking:

- §4.4 user-abort race policy — no conformance fixture yet (the dispatcher's branch-local-vs-global rules implement the policy correctly; just not regression-tested).
- §5.5 `RETRY` × `release_on_catch: false` — engine-side guard described in the runtime plan's self-review; may need explicit code if a fixture surfaces a deadlock.
- `CATCH_LATE_RETURN` log entry — race-condition test; out of fixture scope for v1.
