# TRY / CATCH / RETURN — Design Specification

**Date:** 2026-05-31
**Status:** Approved design (pre-implementation)
**Touches:** Trajectory Editor, Trajectory Runtime
**Does NOT touch:** Action Container REST protocol, Action Container internals, Action Tester

## 0. Locked design decisions

| Decision | Outcome |
|---|---|
| Symbol shapes | CATCH = wide-top trapezoid (inverted); RETURN = narrow-top trapezoid (upright). Footprint matches START / END per notation. |
| TRY scope (v1) | `ACTION PROXY` and `WAIT ACTION PROXY` only. Schema generic for future extension to `SCRIPT` / `WORKFLOW PROXY`. |
| Catch-network containment | Strict island. No edges between main-flow nodes and catch-network nodes. Only inter-network links are TRY → catch_id and the RETURN command. |
| TRY shape | Array of single-mode entries on each step: `[{mode, catch_id, release_on_catch}, …]`. Each `mode` appears at most once per step; different `mode`s on one step may target the same `catch_id`. |
| RETRY semantics | Immediate re-invoke. No built-in retry counter. Author counts via a Value Property + SELECT 1 inside the catch network. |
| RESTART semantics | Has a required flag `restart_mode ∈ {CLEAN, KEEP}`. Both are workflow-global tear-down + relaunch from START. CLEAN resets Value Properties to defaults; KEEP preserves them. |
| Resource cleanup on failure | Per-TRY `release_on_catch: boolean`. Default `true` — failed step's resources release before CATCH activates. |
| Trigger info exposure | CATCH carries user-declared `output_parameter_specifications`. Author binds `trigger_step`, `trigger_step_oid`, `trigger_reason`, `error_message` to Value Properties of their choice. |
| GOTO scope | Targets any main-flow step. Never a catch-network node. Validator enforces. |
| Nested TRY | Allowed. An ACTION PROXY inside a catch network can have its own TRY → a different CATCH. |
| Workflow-global vs branch-local | `ABANDON` and `RESTART` are workflow-global. `RETRY`, `GOTO`, and `COMPLETE` are branch-local (other parallel branches continue unaffected). |
| COMPLETE semantics | Marks the triggering step `COMPLETED` and resumes its own successors as if it had succeeded. Strictly branch-local — never touches a parallel step. No sub-fields. Produces no action outputs (the action failed); the catch network sets Value Properties for any downstream data. |
| Failure classification | Runtime-side, on AC's existing wire format. No Action Container protocol change. |
| Schema version | Stays at `4.0`. Change is pure additive — old packages remain valid; old runtimes reject new packages with the existing `INVALID_STEP_TYPE` error. |
| Catch re-entry while active | Default policy: workflow → `ABORTED` with `CATCH_REENTRY` error. Open to softer policy later. |
| User-abort interaction | User-pressed-abort on the whole workflow lets in-flight TRY ON ABORT catches race teardown. First RETURN command wins; later non-ABANDON RETURNs become no-ops with a `CATCH_LATE_RETURN` log entry. |

---

## 1. Schema additions to `MasterWorkflowStep`

Two new `step_type` enum values: `"CATCH"` and `"RETURN"`. One new optional array `try_specifications` on every step, semantically valid only on `ACTION PROXY` / `WAIT ACTION PROXY` in v1 (validator enforces).

### 1.1 `try_specifications` on action steps

```json
{
  "step_type": "ACTION PROXY",
  ...,
  "resource_command_specifications": [ ... ],
  "try_specifications": [
    { "mode": "ERROR",   "catch_id": "OvenFailed",      "release_on_catch": true  },
    { "mode": "TIMEOUT", "catch_id": "OvenFailed",      "release_on_catch": true  },
    { "mode": "ABORT",   "catch_id": "OperatorAborted", "release_on_catch": false }
  ]
}
```

Field rules:

- `mode ∈ {"ERROR", "ABORT", "TIMEOUT"}`. Required.
- `catch_id` (string). Required. Must resolve to a CATCH step's `catch_id` in the same workflow (validator: `UNMATCHED_TRY`).
- `release_on_catch` (boolean). Optional. Default `true`.
- Within one step, each `mode` value may appear at most once (validator: `DUPLICATE_TRY_MODE`).

### 1.2 CATCH step

```json
{
  "step_type":  "CATCH",
  "local_id":   "Oven Failed Handler",
  "oid":        "300000000000000001",
  "version":    "1.0.0",
  "last_modified_date": "2026-05-31T12:00:00.000Z",
  "description":"Recovers from oven controller faults",
  "position":   { "x": 100, "y": 400 },

  "catch_id":   "OvenFailed",

  "output_parameter_specifications": [
    { "id": "trigger_step",   "target": "FailureContext.StepName" },
    { "id": "trigger_reason", "target": "FailureContext.Mode"     },
    { "id": "error_message",  "target": "FailureContext.Message"  }
  ]
}
```

Field rules:

- All `ManagedElement` fields (`local_id`, `oid`, `version`, `last_modified_date`) required as on any step.
- `catch_id` (string). Required. Unique within the workflow (validator: `DUPLICATE_CATCH_ID`).
- `output_parameter_specifications` (array). Optional. Catch-network authors bind the runtime-supplied trigger info into Value Properties via the existing `target` mechanism.

Runtime-supplied `id` values the author may bind to:

| `id` | Type | Value at CATCH activation |
|---|---|---|
| `trigger_step` | string | `local_id` of the step whose TRY fired. |
| `trigger_step_oid` | string | OID of that step. |
| `trigger_reason` | string | `"ERROR"` / `"ABORT"` / `"TIMEOUT"`. |
| `error_message` | string | AC's `error.message` field, or empty string if null. |

If the author omits `output_parameter_specifications`, the runtime still captures the trigger info in its `active_catches` map for routing purposes — the values just aren't exposed to the workflow's Value Properties.

### 1.3 RETURN step

```json
{
  "step_type":      "RETURN",
  "local_id":       "Restart Workflow",
  "oid":            "300000000000000099",
  "version":        "1.0.0",
  "last_modified_date": "2026-05-31T12:00:00.000Z",
  "position":       { "x": 300, "y": 700 },

  "return_config": {
    "command":       "RESTART",
    "restart_mode":  "KEEP",
    "goto_step_oid": null
  }
}
```

Field rules:

- `return_config.command ∈ {"ABANDON", "RESTART", "GOTO", "RETRY", "COMPLETE"}`. Required.
- `restart_mode ∈ {"CLEAN", "KEEP"}`. Required iff `command == "RESTART"`. No default — the author must choose explicitly. Validator surfaces `MISSING_RESTART_MODE` if absent.
- `goto_step_oid` (string). Required iff `command == "GOTO"`. Must resolve to a main-flow step's `oid` (validator: `GOTO_TARGET_NOT_FOUND`, `GOTO_TARGET_IN_CATCH`).
- `ABANDON`, `RETRY`, and `COMPLETE` use no sub-fields.

### 1.4 Worked example (round-trip)

A workflow with a single TRY catching `ERROR` and `TIMEOUT` of an action, recovering via a USER_INTERACTION acknowledgement, and RETRYing:

```json
{
  "schemaVersion": "4.0",
  "local_id": "Bake With Recovery",
  "oid": "wf-001",
  "version": "1.0.0",
  "last_modified_date": "2026-05-31T12:00:00.000Z",
  "steps": [
    { "local_id": "Start", "oid": "s1", "step_type": "START",
      "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
      "position": { "x": 200, "y": 50 } },
    { "local_id": "Heat Oven", "oid": "s2", "step_type": "ACTION PROXY",
      "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
      "position": { "x": 200, "y": 150 },
      "try_specifications": [
        { "mode": "ERROR",   "catch_id": "OvenFailed", "release_on_catch": true },
        { "mode": "TIMEOUT", "catch_id": "OvenFailed", "release_on_catch": true }
      ]
    },
    { "local_id": "End",   "oid": "s3", "step_type": "END",
      "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
      "position": { "x": 200, "y": 250 } },

    { "local_id": "Oven Failed Handler", "oid": "c1", "step_type": "CATCH",
      "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
      "position": { "x": 500, "y": 50 },
      "catch_id": "OvenFailed",
      "output_parameter_specifications": [
        { "id": "trigger_reason", "target": "FailureContext.Mode" },
        { "id": "error_message",  "target": "FailureContext.Message" }
      ]
    },
    { "local_id": "Acknowledge", "oid": "c2", "step_type": "USER_INTERACTION",
      "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
      "position": { "x": 500, "y": 150 } },
    { "local_id": "Retry", "oid": "c3", "step_type": "RETURN",
      "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
      "position": { "x": 500, "y": 250 },
      "return_config": { "command": "RETRY" }
    }
  ],
  "connections": [
    { "from_step_id": "s1", "to_step_id": "s2" },
    { "from_step_id": "s2", "to_step_id": "s3" },
    { "from_step_id": "c1", "to_step_id": "c2" },
    { "from_step_id": "c2", "to_step_id": "c3" }
  ],
  "value_property_specifications": [
    { "name": "FailureContext",
      "entries": [
        { "name": "Mode",    "value": "" },
        { "name": "Message", "value": "" }
      ] }
  ]
}
```

Catch network (oids `c1`, `c2`, `c3`) is a connected island. No edge crosses to the main flow (`s1`–`s3`). On a failure of `s2`, the runtime activates `c1`, writes `FailureContext.Mode` / `FailureContext.Message`, and the catch network runs through to `c3`'s RETRY, re-invoking `s2`.

---

## 2. Editor UI additions

### 2.1 Palette

Two new draggable items appended to `StepPalette.tsx`'s `stepTypes` and `allStepTypes` arrays:

| `type` (editor internal) | Label | Family |
|---|---|---|
| `catch` | `Catch` | Control-flow entry (same family as `start`) |
| `return` | `Return` | Control-flow exit (same family as `end`) |

### 2.2 Node renderers

Three notation registries get the two new node types. Per-notation dimensions match the START / END footprint:

| Notation | START dimensions | CATCH / RETURN |
|---|---|---|
| flowchart | rounded-rect ~120×50 | trapezoid 120×50 |
| BPMN | circle ~50 diameter | trapezoid 120×50 (BPMN has no native exception-start shape that fits cleanly) |
| ISA-88 PFC | rounded-rect | trapezoid 120×50 |

Two new component files: `CatchNode.tsx`, `ReturnNode.tsx`. Each is registered in `flowchartNodeTypes`, `bpmnNodeTypes`, `isa88NodeTypes` in `src/components/nodes/renderers/{flowchart,bpmn,isa88}/index.ts`.

Handles:

- `CatchNode` exposes a single `Handle position="bottom"` (source). No target handle.
- `ReturnNode` exposes a single `Handle position="top"` (target). No source handle.

Fill conventions (suggested):

- CATCH: light orange fill (`#ffe0b2`), darker orange stroke (`#e65100`). Matches the "warning / recovery" semantics.
- RETURN: light coral fill (`#ffccbc`), darker coral stroke (`#bf360c`).

### 2.3 Connection draw rules

Enforced in the existing `isValidConnection` React Flow callback:

- CATCH rejects any incoming connection.
- RETURN rejects any outgoing connection.
- Any edge whose endpoints are in different partitions (main-flow vs catch-network, computed live via the partition algorithm in §6.5) is rejected at draw time with a toast.

### 2.4 PropertyPanel additions

`NodeInlineEditor.tsx` registers the two new node types:

- `catch` → added to `PARAMETERIZED_TYPES` (renders the `output_parameter_specifications` editor). Also renders a small block with the `catch_id` input and the description editor.
- `return` → added to `CONTROL_FLOW_TYPES`. Renders a new `ReturnConfigEditor` (command selector + conditional sub-fields).

A new section appears on `actionProxy` / `actionProxyWait` nodes, rendered **after** Resource Commands per the user spec: a new `TryEditor` component, mirroring the `ResourceCommandEditor` pattern. UI shape:

```
TRY · error handling                          [+ Add TRY]
┌──────────────────────────────────────────────────────┐
│ ON  [ERROR ▾]   CATCH  [OvenFailed ▾]            [✕] │
│ ☑ Release acquired resources when CATCH activates    │
├──────────────────────────────────────────────────────┤
│ ON  [TIMEOUT ▾] CATCH  [OvenFailed ▾]            [✕] │
│ ☑ Release acquired resources when CATCH activates    │
├──────────────────────────────────────────────────────┤
│ ON  [ABORT ▾]   CATCH  [OperatorAborted ▾]       [✕] │
│ ☐ Release acquired resources when CATCH activates    │
└──────────────────────────────────────────────────────┘
```

UI rules:

- The CATCH dropdown is populated by scanning the workflow for all CATCH steps' `catch_id` values, in alphabetical order.
- The mode dropdown for any row auto-filters to hide modes already used elsewhere on the same step (edit-time enforcement of the mode-uniqueness rule).
- The `release_on_catch` checkbox defaults to checked on row creation.

### 2.5 Change-propagation rules

- Renaming a CATCH's `catch_id` auto-updates every TRY in the workflow that referenced the old id. Single undo entry.
- Deleting a CATCH does NOT silently remove TRY entries that referenced it. Those entries become `UNMATCHED_TRY` validation errors and surface as red chips on the action step. The author must explicitly fix them.
- Deleting a RETURN raises `CATCH_WITHOUT_RETURN` on its catch network until another RETURN is added (or the CATCH is also deleted).

### 2.6 Editor envelope mapping additions

Two new node-type mappings in `NODE_TYPE_TO_STEP_TYPE`:

| Editor `node.type` / `stepType` | Runtime `step_type` |
|---|---|
| `catch` | `CATCH` |
| `return` | `RETURN` |

New `node.data.*` → step-field mappings:

| Editor field | Runtime field | On nodes |
|---|---|---|
| `node.data.trySpecifications` | `try_specifications` | actionProxy, actionProxyWait |
| `node.data.catchId` | `catch_id` | catch |
| `node.data.returnConfig` | `return_config` | return |

---

## 3. Runtime state-machine extensions

### 3.1 New step behavior classes

CATCH and RETURN join `isAutoCompleting()` — they transition `INACTIVE → ACTIVE → COMPLETED` with no `EXECUTING` phase, like START and END.

- CATCH activation: runtime writes its declared `output_parameter_specifications` (the trigger info), then activates its single outgoing connection. Behaviorally identical to a START fire from that node onwards.
- RETURN activation: runtime executes `return_config.command` (full dispatch in §5), then marks RETURN itself COMPLETED.

### 3.2 Per-workflow-instance catch context

A new mutable map lives on the `WorkflowEngine` instance:

```ts
active_catches: Map<string /* catch_oid */, CatchContext>

interface CatchContext {
  catch_oid:              string;
  trigger_step_oid:       string;
  trigger_step_name:      string;     // local_id snapshot, immutable
  trigger_reason:         "ERROR" | "ABORT" | "TIMEOUT";
  error_message:          string | null;
  trigger_input_snapshot: Array<{ name: string; value: string }>;  // audit only
  released_resources:     Array<{ resource_name: string; source_oid: string }>;
  activated_at:           string;     // ISO 8601
}
```

Lifecycle:

- Entry added when CATCH activates.
- Entry removed when its catch network's RETURN executes.
- Entry persisted as part of the workflow-instance row's runtime-state JSON column (existing field).
- At most one entry per `catch_oid` at any time. A TRY firing for an already-active `catch_oid` raises `CATCH_REENTRY` and the workflow → `ABORTED`.

### 3.3 Concurrency under PARALLEL

When PARALLEL fans out two branches and one branch's ACTION PROXY fails:

- Deactivate the failed step. Activate its CATCH. Capture `trigger_step_oid`.
- The other parallel branches keep running unaffected.
- When the catch's RETURN fires:
  - `RETRY`, `GOTO`, and `COMPLETE` are **branch-local** — only the failed branch is touched; other branches continue.
  - `RESTART` (`CLEAN` or `KEEP`) and `ABANDON` are **workflow-global** — all in-flight steps in all branches are torn down. See §5 for exact teardown order.

### 3.4 Engine touch points

- `engine.ts` `activateStep(step)` learns two new step types and delegates to handlers in `step-handlers.ts`.
- The existing AC-terminal consumer (`onActionInstanceTerminal(instance_id, terminal_state, error)`) gains the classification + TRY-dispatch path in §4.
- No new database tables. Persistence rides the existing workflow-instance row's runtime-state JSON column.

---

## 4. Failure-mode classification

The Action Container's wire format already carries everything needed; no AC protocol change. The runtime classifies on the existing `state.current` + `error` fields plus a runtime-side wall-clock timer.

### 4.1 Runtime-side timeout

At REST-03 invoke, the runtime captures:

```ts
const effective_timeout_ms =
  request_body.timeout_ms                       // explicit override
  ?? (action.timeout_seconds ?? 0) * 1000       // action's declared timeout
  ?? null;                                      // null = no TIMEOUT possible
```

If `effective_timeout_ms` is non-null, the runtime starts a wall-clock timer alongside the invoke. On timer fire:

1. Cancel the timer.
2. Send `POST /instances/{id}/command { command: "ABORT" }` to the AC.
3. Mark a pending TIMEOUT classification for this instance.
4. Wait for the AC's terminal state.

### 4.2 Classification on AC terminal

```
if state == "COMPLETED":
    → success; no TRY check.

elif runtime had pending TIMEOUT marker for this instance:
    → mode = TIMEOUT.

elif runtime sent ABORT or STOP to this instance from its own logic
     (workflow-level user-abort, sibling cancellation, etc.):
    → mode = ABORT.

elif error != null:
    → mode = ERROR.   # action_code itself failed.

else:
    → mode = ABORT.   # AC reached ABORTED/STOPPED spontaneously
                      # with no error field; treat as ABORT.
```

### 4.3 TRY dispatch

After classification, before the existing "mark step COMPLETED/FAILED" path:

```
let step = workflow.step(instance.step_oid);
let matching = step.try_specifications?.find(t => t.mode === mode);

if (matching):
    if (matching.release_on_catch):
        release_step_resources(step);
    let catch_step = workflow.find_catch_by_id(matching.catch_id);
    activate_catch(catch_step, {
        trigger_step_oid:       step.oid,
        trigger_step_name:      step.local_id,
        trigger_reason:         mode,
        error_message:          instance.error?.message ?? null,
        trigger_input_snapshot: instance.inputs,
    });
    deactivate_step(step);
    return;  // do NOT propagate failure to workflow level

// no matching TRY:
existing_failure_path(step, mode, error);  // workflow → ABORTED, etc.
```

### 4.4 User-abort interaction

When the user externally aborts the workflow, the runtime sends ABORT to every in-flight ACTION PROXY instance. Each step with `TRY ON ABORT` fires its catch. The catches race the workflow-teardown.

Policy:

- Workflow waits for in-flight catches to reach their RETURN before finalizing teardown.
- First RETURN command wins. If it's `ABANDON`, the workflow confirms the user-abort and tears down.
- Later RETURNs of non-ABANDON commands become no-ops with a `CATCH_LATE_RETURN` log entry.

---

## 5. RETURN command dispatch

At RETURN activation, the runtime executes the matching command, marks RETURN COMPLETED, and removes the matching entry from `active_catches`.

### 5.1 ABANDON (workflow-global)

1. For every active step in the workflow (main flow + every other catch network):
   - Send `ABORT` to its action instance, if any.
   - Mark step `INACTIVE`.
2. Release every workflow-held resource.
3. Workflow state → `ABORTED`.
4. The workflow-instance row's `reason` field records `"ABANDON via CATCH <catch_id>"`.

Reuses the existing `ABORTED` workflow state.

### 5.2 RESTART CLEAN (workflow-global, hard reset)

1. Tear down every active step as in ABANDON, but without setting `ABORTED`.
2. Release every workflow-held resource.
3. Reset every Value Property to its declared `value_property_specifications[*].entries[*].value`.
4. Clear every entry from `active_catches`.
5. Activate START as if the workflow had just launched.
6. Workflow keeps the same `workflow_instance_id`. A new entry is appended to the workflow-instance row's `restart_history`.

Starting parameters are NOT re-prompted — the originals from the launch are used.

### 5.3 RESTART KEEP (workflow-global, soft reset)

Same as CLEAN except step 3 is skipped. Value Properties keep whatever the catch network last wrote. Resources still released — the workflow restarts from START and any Acquire steps run again.

### 5.4 GOTO `<goto_step_oid>` (branch-local)

1. Look up `CatchContext.trigger_step_oid` (the step that originally failed).
2. Mark the trigger step `INACTIVE` if it isn't already (it usually is — TRY made it inactive when CATCH activated).
3. Mark the named GOTO target step `ACTIVE`. The target must be a main-flow step (validator already enforced).
4. Other parallel branches keep running.

If the GOTO target is a `WAIT ALL` / `WAIT ANY` join, it counts as one new arrival at that join from this branch; the existing join logic handles the rest.

### 5.5 RETRY (branch-local)

1. Look up `CatchContext.trigger_step_oid`.
2. Re-resolve the trigger step's `input_parameter_specifications` against the CURRENT Value Property state. The `trigger_input_snapshot` in the catch context is for audit only.
3. Re-acquire any resources the step needs (its `resource_command_specifications` run again).
4. Re-invoke the ACTION PROXY (new `instance_id` from the AC).

If the new invocation fails the same way, the same TRY fires again. Loop-breaking is the author's responsibility.

**Interaction with `release_on_catch: false`.** If the failed TRY had `release_on_catch: false`, the trigger step's resources were still held while the catch network ran. On `RETRY`, the runtime does NOT re-execute the step's `resource_command_specifications` Acquire entries — the resources are already held and re-acquiring would deadlock on `binary exclusive use` or fail validation on `countable`. Instead, RETRY re-invokes the ACTION PROXY directly against the still-held resources. Release entries on the step's `resource_command_specifications` run normally at successful step completion. If the author's intent is "drop resources during recovery and re-acquire before retrying," they MUST set `release_on_catch: true` and rely on the step's Acquire commands to re-acquire on RETRY.

### 5.6 COMPLETE (branch-local)

1. Look up `CatchContext.trigger_step_oid` — the step that originally failed. It is currently `INACTIVE` (TRY deactivated it when the CATCH activated; see §5.4 step 2).
2. Mark the trigger step `COMPLETED` and append the trace entry. Its `EXECUTING → INACTIVE` caught-failure history remains.
3. Activate the trigger step's normal outgoing connection(s) — its success-path successors — as if it had completed. The trigger's completion-time **Release** `resource_command_specifications` run as part of normal completion; a `WAIT ALL` / `WAIT ANY` successor counts as one arrival at that join from this branch (identical to §5.4).
4. Other parallel branches keep running, untouched. `COMPLETE` never advances, completes, aborts, or resets a sibling step. (A shared `WAIT ALL` join still blocks until the siblings arrive on their own.)

No action outputs are produced (the action failed). `COMPLETE` carries no sub-fields. §5.7 cleanup (below) and §4.4 (the `CATCH_LATE_RETURN` no-op rule for non-`ABANDON` commands) apply unchanged.

### 5.7 Catch-network cleanup after any command

- Every step inside the catch network that's `ACTIVE` or `COMPLETED` for this catch run is reset to `INACTIVE`. Catch networks are re-entrant across separate firings; the same network may need to run again later.
- `active_catches[catch_oid]` is removed.
- Resources still held by catch-network steps at RETURN time are released.

### 5.8 Summary

| Command | Workflow state after | Trigger step | Other branches | Resources | Properties |
|---|---|---|---|---|---|
| `ABANDON` | `ABORTED` | torn down | torn down | all released | preserved (audit) |
| `RESTART CLEAN` | running (from START) | torn down | torn down | all released | reset to defaults |
| `RESTART KEEP` | running (from START) | torn down | torn down | all released | preserved |
| `GOTO X` | running (now at X) | inactive | unaffected | trigger's per `release_on_catch` | preserved |
| `RETRY` | running (trigger re-invoking) | re-invoked with current inputs | unaffected | re-acquired per step config | preserved |
| `COMPLETE` | running (past the trigger) | marked `COMPLETED`, successors activated | unaffected | trigger's Release commands run on completion | preserved |

---

## 6. Validator rules

All new checks follow the existing `SCREAMING_SNAKE_CASE` error code convention. Each runs at editor save-time AND at runtime package-load.

### 6.1 Structural (single step)

| Code | Trigger |
|---|---|
| `CATCH_WRONG_DEGREE` | CATCH step has any incoming connection, or ≠1 outgoing connection. |
| `RETURN_WRONG_DEGREE` | RETURN step has ≠1 incoming connection, or any outgoing connection. |
| `TRY_ON_INVALID_STEP` | `try_specifications` non-empty on any step whose `step_type` ∉ `{ACTION PROXY, WAIT ACTION PROXY}`. |
| `INVALID_TRY_MODE` | A `try_specifications[i].mode` ∉ `{ERROR, ABORT, TIMEOUT}`. |
| `DUPLICATE_TRY_MODE` | The same `mode` appears more than once in one step's `try_specifications`. |
| `INVALID_RETURN_COMMAND` | `return_config.command` ∉ `{ABANDON, RESTART, GOTO, RETRY, COMPLETE}`. |
| `MISSING_RESTART_MODE` | `command == "RESTART"` and `restart_mode` is absent. |
| `INVALID_RESTART_MODE` | `restart_mode` present and ∉ `{CLEAN, KEEP}`. |
| `MISSING_GOTO_TARGET` | `command == "GOTO"` and `goto_step_oid` is absent. |

### 6.2 Cross-reference

| Code | Trigger |
|---|---|
| `DUPLICATE_CATCH_ID` | Two CATCH steps in the same workflow share a `catch_id`. |
| `UNMATCHED_TRY` | `try_specifications[i].catch_id` does not resolve to any CATCH in the workflow. |
| `GOTO_TARGET_NOT_FOUND` | `goto_step_oid` does not match any step `oid`. |
| `GOTO_TARGET_IN_CATCH` | `goto_step_oid` resolves to a step inside any catch network. |

### 6.3 Topology

These run after the validator computes the partition `{main_flow_steps, catch_network_steps}`:

| Code | Severity | Trigger |
|---|---|---|
| `CROSS_NETWORK_EDGE` | error | A connection has `from_step` in one partition and `to_step` in the other. |
| `CATCH_WITHOUT_RETURN` | error | A CATCH's downstream subgraph contains no RETURN. |
| `RETURN_WITHOUT_CATCH` | error | A RETURN is not reachable from any CATCH via catch-network edges. |
| `CATCH_NETWORK_NOT_CONNECTED` | error | The catch-network subgraph rooted at a CATCH has internal disconnected components. |
| `ORPHANED_CATCH` | warning | A CATCH whose `catch_id` is not referenced by any TRY. Editor renders a yellow chip on the node; runtime accepts. |

Multiple RETURNs per CATCH network are allowed.

### 6.4 Runtime-only dynamic checks

| Code | Trigger |
|---|---|
| `CATCH_REENTRY` | A TRY fires for a `catch_id` whose entry is already in `active_catches`. Workflow → `ABORTED`. |
| `CATCH_LATE_RETURN` | A RETURN command other than `ABANDON` fires after another RETURN has already issued `ABANDON` during user-abort teardown. Logged, RETURN is a no-op. |

### 6.5 Partition algorithm

```
catch_network_steps = ∅
for each CATCH step c in workflow.steps:
    visited = BFS over outgoing connections from c
              (treating each visited step as a frontier)
    if visited contains at least one RETURN:
        catch_network_steps ∪= visited
main_flow_steps = workflow.steps \ catch_network_steps
```

The partition is consumed by `CROSS_NETWORK_EDGE`, `GOTO_TARGET_IN_CATCH`, and the runtime's TRY-dispatch path. Edge cases caught earlier by structural-degree checks fire first.

### 6.6 Editor-time vs runtime-time

The editor runs every rule on every change (live-validation panel chip + node-level red dot). The runtime runs every rule once at package-load. Code is shared if the validator module is shared between repos; otherwise the rules are mirrored.

The runtime additionally enforces the two dynamic checks in §6.4.

---

## 7. Migration, compatibility, fixtures, docs

### 7.1 Schema version status

Stays at `schemaVersion: "4.0"`. The change is pure superset:

- New optional `try_specifications` array on existing step types.
- New `step_type` enum values `"CATCH"` and `"RETURN"`.
- New `catch_id` and `return_config` fields on those new step types.

No removed fields. No changed field types. Old `.WFmaster*` files load unchanged.

### 7.2 Backward compatibility (new runtime ↔ old packages)

A new runtime encountering a pre-TRY package sees zero `try_specifications` arrays and no CATCH / RETURN steps. The new failure-classification code in §4 only matters when a matching TRY exists. Without a TRY, the existing "workflow → error" path fires as today.

Net: zero regression risk for existing workflows.

### 7.3 Forward compatibility (old runtime ↔ new packages)

The runtime's AJV-validation step rejects unknown step types with the existing `INVALID_STEP_TYPE` error. That's the right behavior — the workflow uses control flow the runtime can't execute.

No version bump because the schema enum already gates this.

### 7.4 Conformance fixtures to add

Pattern: existing `TrajectoryRuntime/spec/conformance/execution/exec-<feature>-<NNN>-<name>.json` shape with `test_id`, `name`, `category`, `tags`, `workflow`, `expected.execution_trace`, `expected.workflow_state`.

Execution fixtures:

| # | Filename | Asserts |
|---|---|---|
| 1 | `exec-try-catch-001-error-to-catch.json` | ACTION PROXY fails with `ERROR`; CATCH activates; RETURN `ABANDON`; workflow → `ABORTED`. |
| 2 | `exec-try-catch-002-timeout-to-retry.json` | ACTION PROXY times out; CATCH activates; RETURN `RETRY`; second invocation succeeds. |
| 3 | `exec-try-catch-003-abort-to-goto.json` | Engine `ABORT`s an ACTION PROXY; CATCH activates; RETURN `GOTO X`; main flow resumes at X. |
| 4 | `exec-try-catch-004-restart-keep.json` | Property mutated in catch network; RETURN `RESTART KEEP`; main flow re-runs with mutated property. |
| 5 | `exec-try-catch-005-restart-clean.json` | Same as 004 but `RESTART CLEAN`; property reset to default. |
| 6 | `exec-try-catch-006-multi-mode-mapping.json` | One step has `TRY ON ERROR → C1`, `TRY ON TIMEOUT → C1`, `TRY ON ABORT → C2`; verify correct routing per induced failure. |
| 7 | `exec-try-catch-007-parallel-branch-local.json` | `PARALLEL` fans out two ACTION PROXIES; one fails and `RETRY`s; other continues unaffected. |
| 8 | `exec-try-catch-008-error-to-complete.json` | ACTION PROXY fails with `ERROR`; CATCH activates; RETURN `COMPLETE`; trigger force-completed; main flow resumes → `COMPLETED`. |
| 9 | `exec-try-catch-009-release-on-catch-true.json` | Failed step held a resource; CATCH activates with `release_on_catch: true`; resource available again in catch network. |
| 10 | `exec-try-catch-010-release-on-catch-false.json` | Failed step held a resource; CATCH with `release_on_catch: false`; catch network sees resource still held. |

Validator fixtures (`spec/conformance/validation/`):

| # | Filename | Asserts |
|---|---|---|
| V1 | `valid-cross-network-edge.json` | Rejects with `CROSS_NETWORK_EDGE`. |
| V2 | `valid-catch-without-return.json` | Rejects with `CATCH_WITHOUT_RETURN`. |
| V3 | `valid-duplicate-catch-id.json` | Rejects with `DUPLICATE_CATCH_ID`. |
| V4 | `valid-unmatched-try.json` | Rejects with `UNMATCHED_TRY`. |
| V5 | `valid-goto-into-catch.json` | Rejects with `GOTO_TARGET_IN_CATCH`. |
| V6 | `valid-try-on-script.json` | Rejects with `TRY_ON_INVALID_STEP` (until the v1 enum expands). |
| V7 | `valid-orphaned-catch-warning.json` | Loads successfully; warning surfaced in validator output but `valid: true`. |

### 7.5 Documentation updates

Three docs touched in the umbrella repo:

- `docs/JSON_SCHEMA.md` — extend Part 1: add `CATCH` and `RETURN` to the step_type enum (§1.5.1); add `try_specifications` to the per-step fields (§1.5.7); add new subsections covering CATCH config and RETURN config. Extend Part 4 (editor envelope → runtime envelope) with the new node-type mapping rows.
- `docs/REST_Protocol.md` — no protocol change; add a sentence in §3.3 (opaque vs observable) noting the runtime classifies AC terminals into `ERROR/ABORT/TIMEOUT` for catch dispatch (informational; doesn't change the AC contract).
- New: `TrajectoryRuntime/spec/docs/error-handling.md` — runtime-side reference for implementers. This spec is the source.

---

## Appendix A — Full validator error code inventory

| Code | Severity | Origin |
|---|---|---|
| `CATCH_WRONG_DEGREE` | error | structural |
| `RETURN_WRONG_DEGREE` | error | structural |
| `TRY_ON_INVALID_STEP` | error | structural |
| `INVALID_TRY_MODE` | error | structural |
| `DUPLICATE_TRY_MODE` | error | structural |
| `INVALID_RETURN_COMMAND` | error | structural |
| `MISSING_RESTART_MODE` | error | structural |
| `INVALID_RESTART_MODE` | error | structural |
| `MISSING_GOTO_TARGET` | error | structural |
| `DUPLICATE_CATCH_ID` | error | cross-reference |
| `UNMATCHED_TRY` | error | cross-reference |
| `GOTO_TARGET_NOT_FOUND` | error | cross-reference |
| `GOTO_TARGET_IN_CATCH` | error | cross-reference |
| `CROSS_NETWORK_EDGE` | error | topology |
| `CATCH_WITHOUT_RETURN` | error | topology |
| `RETURN_WITHOUT_CATCH` | error | topology |
| `CATCH_NETWORK_NOT_CONNECTED` | error | topology |
| `ORPHANED_CATCH` | warning | topology |
| `CATCH_REENTRY` | runtime error | dynamic |
| `CATCH_LATE_RETURN` | runtime log | dynamic |

## Appendix B — Files touched, by repo

### `TrajectoryEditor`

| File | Change |
|---|---|
| `src/types/nodes.ts` | Add `'catch'` and `'return'` to `NodeType` union. |
| `src/components/nodes/renderers/flowchart/index.ts` | Register `CatchNode`, `ReturnNode`. |
| `src/components/nodes/renderers/bpmn/index.ts` | Register `CatchNode`, `ReturnNode`. |
| `src/components/nodes/renderers/isa88/index.ts` | Register `CatchNode`, `ReturnNode`. |
| `src/components/nodes/renderers/flowchart/CatchNode.tsx` | New. |
| `src/components/nodes/renderers/flowchart/ReturnNode.tsx` | New. |
| `src/components/nodes/renderers/notation-config.ts` | Add trapezoid dimensions per notation. |
| `src/components/palette/StepPalette.tsx` | Add `catch`, `return` to `stepTypes` and `allStepTypes`. |
| `src/components/properties/NodeInlineEditor.tsx` | Add `catch` to `PARAMETERIZED_TYPES`; `return` to `CONTROL_FLOW_TYPES`. |
| `src/components/properties/TryEditor.tsx` | New. |
| `src/components/properties/ReturnConfigEditor.tsx` | New. |
| `src/components/properties/CatchIdEditor.tsx` | New (small input + description). |
| `src/lib/packageFormat/export-types.ts` | Add `catch: 'CATCH'`, `return: 'RETURN'` to `NODE_TYPE_TO_STEP_TYPE`. |
| `src/lib/packageFormat/step-extractors.ts` | Add `extractTrySpecifications`, `extractReturnConfig`. |
| `src/lib/packageFormat/workflow-exporters.ts` | Wire the new extractors into the per-step emit path. |
| `src/lib/import/workflow-importers.ts` | Read `try_specifications`, `catch_id`, `return_config` into `node.data.*`. |
| `src/lib/validation/*` | Add the structural / cross-reference / topology checks. |
| `src/store/specStore.ts` | Add catch_id rename-propagation logic. |

### `TrajectoryRuntime`

| File | Change |
|---|---|
| `spec/workflow-schema.json` | Add `CATCH`, `RETURN` to `step_type` enum; add `TryConfig`, `CatchConfig`, `ReturnConfig` to `$defs`; add the new optional fields. |
| `engines/web/src/types.ts` | Add `CatchContext` interface. |
| `engines/web/src/step-handlers.ts` | Add CATCH / RETURN to `isAutoCompleting()`; add CATCH activation handler (writes output_parameter_specifications). |
| `engines/web/src/engine.ts` | Add `active_catches` map; add classification + TRY-dispatch in AC-terminal consumer; add RETURN command dispatch; add runtime-side timeout. |
| `engines/web/src/validator.ts` | Add every rule in §6.1–§6.3. |
| `engines/web/src/loader.ts` | No change (the loader only reads JSON; new fields ride through). |
| `spec/conformance/execution/exec-try-catch-001…010.json` | New fixtures. |
| `spec/conformance/validation/valid-*.json` | New fixtures (V1–V7). |
| `spec/docs/error-handling.md` | New. |

### `Trajectory` (umbrella docs repo)

| File | Change |
|---|---|
| `docs/JSON_SCHEMA.md` | Add CATCH / RETURN / try_specifications coverage. |
| `docs/REST_Protocol.md` | One-paragraph mention of runtime-side failure classification. |

## Appendix C — Open items deferred to implementation

These are intentionally NOT decided in this spec; the implementer chooses:

- Exact React Flow handle id strings for the new node types (any internally-consistent choice works).
- The literal SVG path strings for the trapezoid per notation (the §2.2 dimensions and the funnel orientation locked in §0 are the only constraints).
- Whether the validator module is duplicated or shared between editor and runtime (existing project pattern dictates).
- The exact wire format of `restart_history` entries inside the workflow-instance row's runtime-state JSON (existing project pattern dictates).
- Telemetry / logging fields beyond the named error codes.
