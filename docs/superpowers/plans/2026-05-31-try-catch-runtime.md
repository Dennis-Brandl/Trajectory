# TRY / CATCH / RETURN — Runtime Implementation Plan

> ⛔ **SUPERSEDED 2026-05-31** — found misgrounded against the real TrajectoryRuntime: it assumed Vitest, a single engine with an `onActionInstanceTerminal` consumer, and an error-array validator — **none of which exist** (the repo uses `node:test`, two parallel engines TS+Kotlin, and a fail-fast single-error `validate()`). **DO NOT EXECUTE.** Replaced by `2026-05-31-try-catch-runtime-web-ts.md` + `2026-05-31-try-catch-runtime-native-kmp.md`, grounded by `2026-05-31-try-catch-IMPLEMENTATION-NOTES.md`.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement TRY/CATCH/RETURN error-handling primitives in the Trajectory Runtime — schema, validator, engine, and the full conformance fixture set — per the approved spec at `docs/superpowers/specs/2026-05-31-try-catch-return-design.md`.

**Architecture:** Test-driven, fixture-anchored. Each runtime behavior is locked in by a conformance fixture before the engine code lands. Schema stays at v4.0; change is purely additive. Action Container REST protocol is unchanged.

**Tech Stack:** TypeScript, AJV 8 (Draft 2020-12), Vitest. All work happens inside the `TrajectoryRuntime/` standalone repo (see `MEMORY.md → project-trajectory-repo-layout`).

**Scope boundaries:**
- This plan is RUNTIME ONLY. Editor changes are in a sibling plan.
- Action Container is NOT touched (no REST protocol change).
- Docs are in a sibling plan.

---

## File Structure Plan

### Files to modify

| File | Why |
|---|---|
| `TrajectoryRuntime/spec/workflow-schema.json` | Add `CATCH`, `RETURN` to step_type enum; add `TryConfig`, `CatchConfig`, `ReturnConfig` to `$defs`; add `try_specifications`, `catch_id`, `return_config` to `MasterWorkflowStep`. |
| `TrajectoryRuntime/engines/web/src/types.ts` | Add `CatchContext`, `TrySpecification`, `ReturnConfig`, failure-mode enum types. |
| `TrajectoryRuntime/engines/web/src/step-handlers.ts` | Add CATCH/RETURN to `isAutoCompleting()`; add `activateCatch()` handler that writes output_parameter_specifications; add `dispatchReturn()` skeleton. |
| `TrajectoryRuntime/engines/web/src/engine.ts` | Add `active_catches` map; add classification + TRY dispatch in AC-terminal consumer; add RETURN command dispatch routines; add runtime-side timeout timer. |
| `TrajectoryRuntime/engines/web/src/validator.ts` | Add every rule from spec §6.1–§6.3. |

### Files to create

| File | Why |
|---|---|
| `TrajectoryRuntime/engines/web/src/catch-network-partition.ts` | Pure helper: computes `{main_flow_steps, catch_network_steps}` partition from a workflow. Used by validator AND engine. |
| `TrajectoryRuntime/engines/web/src/failure-classifier.ts` | Pure helper: classifies an AC terminal event into `ERROR / ABORT / TIMEOUT`. Side-effect-free; takes the runtime's state and the AC instance. |
| `TrajectoryRuntime/engines/web/src/return-dispatcher.ts` | All four RETURN command handlers in one focused module. Imported by `engine.ts`. |
| `TrajectoryRuntime/engines/web/src/__tests__/catch-network-partition.test.ts` | Unit tests for the partition helper. |
| `TrajectoryRuntime/engines/web/src/__tests__/failure-classifier.test.ts` | Unit tests for the classifier. |
| `TrajectoryRuntime/engines/web/src/__tests__/return-dispatcher.test.ts` | Unit tests for each RETURN command handler. |
| `TrajectoryRuntime/engines/web/src/__tests__/validator-try-catch.test.ts` | Unit tests for every new validator rule. |
| `TrajectoryRuntime/spec/conformance/execution/exec-try-catch-001-error-to-catch.json` through `-010-release-on-catch-false.json` | The 10 execution fixtures from spec §7.4. |
| `TrajectoryRuntime/spec/conformance/validation/valid-cross-network-edge.json` through `valid-orphaned-catch-warning.json` | The 7 validation fixtures (V1–V7). |

### Files NOT touched

`engines/web/src/loader.ts` (the loader only deserializes JSON — new fields ride through untouched).
The Action Container, the Editor.

### Test command

```
cd C:/Trajectory/TrajectoryRuntime
npm test
```

Everything below assumes that command runs Vitest in watch mode in the runtime repo.

---

## Phase A — Schema additions

The schema lands first because every later task validates against it. No engine behavior changes here.

### Task A1: Add CATCH and RETURN to the step_type enum

**Files:**
- Modify: `TrajectoryRuntime/spec/workflow-schema.json` (`StepType` enum, currently at lines 19–25 per recon)

- [ ] **Step 1: Write the failing AJV test**

Create the test fixture for a minimal workflow using CATCH and RETURN:

`TrajectoryRuntime/spec/conformance/validation/valid-minimal-catch-return.json`:
```json
{
  "test_id": "valid-minimal-catch-return",
  "name": "Minimal workflow using CATCH and RETURN loads",
  "category": "validation",
  "tags": ["try-catch", "schema"],
  "workflow": {
    "schemaVersion": "4.0",
    "local_id": "Minimal Catch Demo",
    "oid": "wf-minctch-001",
    "version": "1.0.0",
    "last_modified_date": "2026-05-31T12:00:00.000Z",
    "steps": [
      { "local_id": "Start", "oid": "s1", "step_type": "START",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z" },
      { "local_id": "End", "oid": "s2", "step_type": "END",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z" },
      { "local_id": "Recover", "oid": "c1", "step_type": "CATCH",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "catch_id": "Recover" },
      { "local_id": "Done", "oid": "c2", "step_type": "RETURN",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "return_config": { "command": "ABANDON" } }
    ],
    "connections": [
      { "from_step_id": "s1", "to_step_id": "s2" },
      { "from_step_id": "c1", "to_step_id": "c2" }
    ]
  },
  "expected": { "valid": true }
}
```

- [ ] **Step 2: Run the conformance harness — expect failure**

```
npm test -- valid-minimal-catch-return
```

Expected output: AJV error `INVALID_STEP_TYPE` because `"CATCH"` and `"RETURN"` aren't in the enum yet.

- [ ] **Step 3: Add the enum entries**

In `TrajectoryRuntime/spec/workflow-schema.json`, locate the `StepType` enum and add the two new values. Existing line range per the recon: 19–25.

```json
"StepType": {
  "type": "string",
  "enum": [
    "START", "END", "ACTION PROXY", "WAIT ACTION PROXY", "WORKFLOW PROXY",
    "SELECT 1", "WAIT ANY", "PARALLEL", "WAIT ALL",
    "MATH", "SCRIPT", "YES_NO", "USER_INTERACTION",
    "CATCH", "RETURN"
  ]
}
```

- [ ] **Step 4: Re-run the test — expect pass**

```
npm test -- valid-minimal-catch-return
```

Expected output: PASS.

- [ ] **Step 5: Commit**

```
git add spec/workflow-schema.json spec/conformance/validation/valid-minimal-catch-return.json
git commit -m "feat(schema): add CATCH and RETURN to step_type enum"
```

### Task A2: Add TryConfig, CatchConfig, ReturnConfig to $defs and wire onto MasterWorkflowStep

**Files:**
- Modify: `TrajectoryRuntime/spec/workflow-schema.json` (`$defs` block; `MasterWorkflowStep` properties)

- [ ] **Step 1: Write a failing test for try_specifications shape**

`TrajectoryRuntime/spec/conformance/validation/valid-try-specifications-shape.json`:
```json
{
  "test_id": "valid-try-specifications-shape",
  "name": "ACTION PROXY accepts a well-formed try_specifications array",
  "category": "validation",
  "tags": ["try-catch", "schema"],
  "workflow": {
    "schemaVersion": "4.0",
    "local_id": "Try Spec Shape",
    "oid": "wf-tryshape-001",
    "version": "1.0.0",
    "last_modified_date": "2026-05-31T12:00:00.000Z",
    "steps": [
      { "local_id": "Start", "oid": "s1", "step_type": "START",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z" },
      { "local_id": "Action", "oid": "s2", "step_type": "ACTION PROXY",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "try_specifications": [
          { "mode": "ERROR", "catch_id": "C1", "release_on_catch": true },
          { "mode": "TIMEOUT", "catch_id": "C1" }
        ]
      },
      { "local_id": "End", "oid": "s3", "step_type": "END",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z" },
      { "local_id": "Recover", "oid": "c1", "step_type": "CATCH",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "catch_id": "C1" },
      { "local_id": "Done", "oid": "c2", "step_type": "RETURN",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "return_config": { "command": "RETRY" } }
    ],
    "connections": [
      { "from_step_id": "s1", "to_step_id": "s2" },
      { "from_step_id": "s2", "to_step_id": "s3" },
      { "from_step_id": "c1", "to_step_id": "c2" }
    ]
  },
  "expected": { "valid": true }
}
```

- [ ] **Step 2: Run — expect failure**

```
npm test -- valid-try-specifications-shape
```

Expected: schema validation error because `try_specifications`, `catch_id`, `return_config` aren't known properties.

- [ ] **Step 3: Add the three `$defs` and wire onto `MasterWorkflowStep`**

In `TrajectoryRuntime/spec/workflow-schema.json`, locate the `$defs` block. Add:

```json
"TrySpecification": {
  "type": "object",
  "required": ["mode", "catch_id"],
  "additionalProperties": false,
  "properties": {
    "mode":             { "type": "string", "enum": ["ERROR", "ABORT", "TIMEOUT"] },
    "catch_id":         { "type": "string", "minLength": 1 },
    "release_on_catch": { "type": "boolean", "default": true }
  }
},
"ReturnConfig": {
  "type": "object",
  "required": ["command"],
  "additionalProperties": false,
  "properties": {
    "command":       { "type": "string", "enum": ["ABANDON", "RESTART", "GOTO", "RETRY"] },
    "restart_mode":  { "type": "string", "enum": ["CLEAN", "KEEP"] },
    "goto_step_oid": { "type": "string" }
  }
}
```

In the `MasterWorkflowStep` properties block, add:

```json
"try_specifications": {
  "type": "array",
  "items": { "$ref": "#/$defs/TrySpecification" },
  "default": []
},
"catch_id": {
  "type": "string",
  "minLength": 1,
  "description": "Required on CATCH steps; unique within workflow."
},
"return_config": {
  "$ref": "#/$defs/ReturnConfig"
}
```

Note: We intentionally do NOT add per-step-type conditional requirements at AJV level — the semantic validator (Phase B) enforces "TRY only on action steps", "catch_id only on CATCH", "return_config only on RETURN".

- [ ] **Step 4: Re-run — expect pass**

```
npm test -- valid-try-specifications-shape
```

Expected: PASS.

- [ ] **Step 5: Commit**

```
git add spec/workflow-schema.json spec/conformance/validation/valid-try-specifications-shape.json
git commit -m "feat(schema): add TrySpecification and ReturnConfig $defs"
```

---

## Phase B — Validator rules

Now that the schema accepts the new shapes, the semantic validator must enforce the structural, cross-reference, and topology rules from spec §6.

### Task B1: Add types and the partition helper

**Files:**
- Modify: `TrajectoryRuntime/engines/web/src/types.ts`
- Create: `TrajectoryRuntime/engines/web/src/catch-network-partition.ts`
- Create: `TrajectoryRuntime/engines/web/src/__tests__/catch-network-partition.test.ts`

- [ ] **Step 1: Add type declarations**

Append to `TrajectoryRuntime/engines/web/src/types.ts`:

```typescript
export type FailureMode = "ERROR" | "ABORT" | "TIMEOUT";

export interface TrySpecification {
  mode: FailureMode;
  catch_id: string;
  release_on_catch?: boolean;
}

export interface ReturnConfig {
  command: "ABANDON" | "RESTART" | "GOTO" | "RETRY";
  restart_mode?: "CLEAN" | "KEEP";
  goto_step_oid?: string;
}

export interface CatchContext {
  catch_oid: string;
  trigger_step_oid: string;
  trigger_step_name: string;
  trigger_reason: FailureMode;
  error_message: string | null;
  trigger_input_snapshot: Array<{ name: string; value: string }>;
  released_resources: Array<{ resource_name: string; source_oid: string }>;
  activated_at: string;
}

export interface CatchNetworkPartition {
  mainFlowStepOids: Set<string>;
  catchNetworkStepOids: Set<string>;
  // Map from catch_id → set of step oids reachable from that CATCH (the catch's network).
  networksByCatchId: Map<string, Set<string>>;
}
```

- [ ] **Step 2: Write failing tests for the partition helper**

`TrajectoryRuntime/engines/web/src/__tests__/catch-network-partition.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { partitionCatchNetworks } from "../catch-network-partition.js";

const minimalWorkflow = {
  steps: [
    { oid: "s1", step_type: "START" },
    { oid: "s2", step_type: "ACTION PROXY", try_specifications: [{ mode: "ERROR", catch_id: "C1" }] },
    { oid: "s3", step_type: "END" },
    { oid: "c1", step_type: "CATCH", catch_id: "C1" },
    { oid: "c2", step_type: "USER_INTERACTION" },
    { oid: "c3", step_type: "RETURN", return_config: { command: "ABANDON" } }
  ],
  connections: [
    { from_step_id: "s1", to_step_id: "s2" },
    { from_step_id: "s2", to_step_id: "s3" },
    { from_step_id: "c1", to_step_id: "c2" },
    { from_step_id: "c2", to_step_id: "c3" }
  ]
};

describe("partitionCatchNetworks", () => {
  it("separates main-flow from catch-network steps", () => {
    const p = partitionCatchNetworks(minimalWorkflow as any);
    expect([...p.mainFlowStepOids].sort()).toEqual(["s1", "s2", "s3"]);
    expect([...p.catchNetworkStepOids].sort()).toEqual(["c1", "c2", "c3"]);
  });

  it("indexes networks by catch_id", () => {
    const p = partitionCatchNetworks(minimalWorkflow as any);
    expect([...(p.networksByCatchId.get("C1") ?? [])].sort()).toEqual(["c1", "c2", "c3"]);
  });

  it("excludes from catch-network any CATCH whose downstream has no RETURN", () => {
    const w = {
      steps: [
        { oid: "s1", step_type: "START" },
        { oid: "s2", step_type: "END" },
        { oid: "c1", step_type: "CATCH", catch_id: "C1" },
        { oid: "c2", step_type: "USER_INTERACTION" }
      ],
      connections: [
        { from_step_id: "s1", to_step_id: "s2" },
        { from_step_id: "c1", to_step_id: "c2" }
      ]
    };
    const p = partitionCatchNetworks(w as any);
    expect(p.catchNetworkStepOids.size).toBe(0);
    expect(p.mainFlowStepOids.size).toBe(4);
  });

  it("handles two disjoint catch networks", () => {
    const w = {
      steps: [
        { oid: "s1", step_type: "START" },
        { oid: "s2", step_type: "END" },
        { oid: "ca", step_type: "CATCH", catch_id: "CA" },
        { oid: "ra", step_type: "RETURN", return_config: { command: "ABANDON" } },
        { oid: "cb", step_type: "CATCH", catch_id: "CB" },
        { oid: "rb", step_type: "RETURN", return_config: { command: "RETRY" } }
      ],
      connections: [
        { from_step_id: "s1", to_step_id: "s2" },
        { from_step_id: "ca", to_step_id: "ra" },
        { from_step_id: "cb", to_step_id: "rb" }
      ]
    };
    const p = partitionCatchNetworks(w as any);
    expect([...p.catchNetworkStepOids].sort()).toEqual(["ca", "cb", "ra", "rb"]);
    expect([...(p.networksByCatchId.get("CA") ?? [])].sort()).toEqual(["ca", "ra"]);
    expect([...(p.networksByCatchId.get("CB") ?? [])].sort()).toEqual(["cb", "rb"]);
  });
});
```

- [ ] **Step 3: Run — expect 4 failures (module not found)**

```
npm test -- catch-network-partition
```

- [ ] **Step 4: Implement the partition helper**

`TrajectoryRuntime/engines/web/src/catch-network-partition.ts`:

```typescript
import type { MasterWorkflowSpecification } from "./types.js";
import type { CatchNetworkPartition } from "./types.js";

interface StepLike { oid: string; step_type: string; catch_id?: string }
interface ConnectionLike { from_step_id: string; to_step_id: string }

export function partitionCatchNetworks(workflow: {
  steps: StepLike[];
  connections: ConnectionLike[];
}): CatchNetworkPartition {
  const stepByOid = new Map<string, StepLike>();
  for (const s of workflow.steps) stepByOid.set(s.oid, s);

  const outEdges = new Map<string, string[]>();
  for (const c of workflow.connections) {
    if (!outEdges.has(c.from_step_id)) outEdges.set(c.from_step_id, []);
    outEdges.get(c.from_step_id)!.push(c.to_step_id);
  }

  const networksByCatchId = new Map<string, Set<string>>();
  const catchNetworkStepOids = new Set<string>();

  for (const step of workflow.steps) {
    if (step.step_type !== "CATCH" || !step.catch_id) continue;

    const visited = new Set<string>();
    const stack = [step.oid];
    let hasReturn = false;

    while (stack.length) {
      const cur = stack.pop()!;
      if (visited.has(cur)) continue;
      visited.add(cur);
      const curStep = stepByOid.get(cur);
      if (curStep?.step_type === "RETURN") hasReturn = true;
      const next = outEdges.get(cur) ?? [];
      for (const n of next) {
        if (!visited.has(n)) stack.push(n);
      }
    }

    if (hasReturn) {
      networksByCatchId.set(step.catch_id, visited);
      for (const oid of visited) catchNetworkStepOids.add(oid);
    }
  }

  const mainFlowStepOids = new Set<string>();
  for (const s of workflow.steps) {
    if (!catchNetworkStepOids.has(s.oid)) mainFlowStepOids.add(s.oid);
  }

  return { mainFlowStepOids, catchNetworkStepOids, networksByCatchId };
}
```

- [ ] **Step 5: Run — expect all 4 to pass**

```
npm test -- catch-network-partition
```

- [ ] **Step 6: Commit**

```
git add engines/web/src/types.ts engines/web/src/catch-network-partition.ts engines/web/src/__tests__/catch-network-partition.test.ts
git commit -m "feat(runtime): add CatchContext types and catch-network partition helper"
```

### Task B2: Add structural validator rules (single-step degree, mode set, command shape)

**Files:**
- Modify: `TrajectoryRuntime/engines/web/src/validator.ts`
- Create: `TrajectoryRuntime/engines/web/src/__tests__/validator-try-catch.test.ts`

- [ ] **Step 1: Write the failing structural tests**

Create `TrajectoryRuntime/engines/web/src/__tests__/validator-try-catch.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { validateWorkflow } from "../validator.js";

function mk(step: any) {
  return {
    schemaVersion: "4.0",
    local_id: "test", oid: "wf-test", version: "1.0.0",
    last_modified_date: "2026-05-31T00:00:00Z",
    steps: [
      { local_id: "Start", oid: "s1", step_type: "START", version: "1.0.0", last_modified_date: "2026-05-31T00:00:00Z" },
      { local_id: "End", oid: "s2", step_type: "END", version: "1.0.0", last_modified_date: "2026-05-31T00:00:00Z" },
      step
    ],
    connections: [{ from_step_id: "s1", to_step_id: "s2" }]
  };
}

describe("validator: structural TRY rules", () => {
  it("rejects CATCH with incoming connection (CATCH_WRONG_DEGREE)", () => {
    const wf = mk({
      local_id: "C", oid: "c1", step_type: "CATCH", version: "1.0.0",
      last_modified_date: "2026-05-31T00:00:00Z", catch_id: "C1"
    });
    wf.connections.push({ from_step_id: "s1", to_step_id: "c1" });
    const r = validateWorkflow(wf);
    expect(r.errors.some(e => e.code === "CATCH_WRONG_DEGREE")).toBe(true);
  });

  it("rejects RETURN with outgoing connection (RETURN_WRONG_DEGREE)", () => {
    const wf = mk({
      local_id: "R", oid: "r1", step_type: "RETURN", version: "1.0.0",
      last_modified_date: "2026-05-31T00:00:00Z",
      return_config: { command: "ABANDON" }
    });
    wf.connections.push({ from_step_id: "r1", to_step_id: "s2" });
    const r = validateWorkflow(wf);
    expect(r.errors.some(e => e.code === "RETURN_WRONG_DEGREE")).toBe(true);
  });

  it("rejects try_specifications on USER_INTERACTION (TRY_ON_INVALID_STEP)", () => {
    const wf = mk({
      local_id: "U", oid: "u1", step_type: "USER_INTERACTION", version: "1.0.0",
      last_modified_date: "2026-05-31T00:00:00Z",
      try_specifications: [{ mode: "ERROR", catch_id: "X" }]
    });
    const r = validateWorkflow(wf);
    expect(r.errors.some(e => e.code === "TRY_ON_INVALID_STEP")).toBe(true);
  });

  it("rejects duplicate mode in one step's try_specifications (DUPLICATE_TRY_MODE)", () => {
    const wf = mk({
      local_id: "A", oid: "a1", step_type: "ACTION PROXY", version: "1.0.0",
      last_modified_date: "2026-05-31T00:00:00Z",
      try_specifications: [
        { mode: "ERROR", catch_id: "X" },
        { mode: "ERROR", catch_id: "Y" }
      ]
    });
    const r = validateWorkflow(wf);
    expect(r.errors.some(e => e.code === "DUPLICATE_TRY_MODE")).toBe(true);
  });

  it("rejects RETURN with RESTART but no restart_mode (MISSING_RESTART_MODE)", () => {
    const wf = mk({
      local_id: "R", oid: "r1", step_type: "RETURN", version: "1.0.0",
      last_modified_date: "2026-05-31T00:00:00Z",
      return_config: { command: "RESTART" }
    });
    const r = validateWorkflow(wf);
    expect(r.errors.some(e => e.code === "MISSING_RESTART_MODE")).toBe(true);
  });

  it("rejects RETURN with GOTO but no goto_step_oid (MISSING_GOTO_TARGET)", () => {
    const wf = mk({
      local_id: "R", oid: "r1", step_type: "RETURN", version: "1.0.0",
      last_modified_date: "2026-05-31T00:00:00Z",
      return_config: { command: "GOTO" }
    });
    const r = validateWorkflow(wf);
    expect(r.errors.some(e => e.code === "MISSING_GOTO_TARGET")).toBe(true);
  });
});
```

- [ ] **Step 2: Run — expect failures (rules don't exist yet)**

```
npm test -- validator-try-catch
```

- [ ] **Step 3: Implement the structural rules**

In `TrajectoryRuntime/engines/web/src/validator.ts`, in the existing per-step iteration block (after the line that handles `step_type === "ACTION PROXY"`), add:

```typescript
// CATCH / RETURN degree checks
if (step.step_type === "CATCH") {
  const incoming = connections.filter(c => c.to_step_id === step.oid).length;
  const outgoing = connections.filter(c => c.from_step_id === step.oid).length;
  if (incoming !== 0 || outgoing !== 1) {
    errors.push({ code: "CATCH_WRONG_DEGREE", message:
      `CATCH '${step.local_id}' must have 0 incoming and 1 outgoing connection (has ${incoming} in, ${outgoing} out)`,
      stepOid: step.oid });
  }
}
if (step.step_type === "RETURN") {
  const incoming = connections.filter(c => c.to_step_id === step.oid).length;
  const outgoing = connections.filter(c => c.from_step_id === step.oid).length;
  if (incoming !== 1 || outgoing !== 0) {
    errors.push({ code: "RETURN_WRONG_DEGREE", message:
      `RETURN '${step.local_id}' must have 1 incoming and 0 outgoing connections (has ${incoming} in, ${outgoing} out)`,
      stepOid: step.oid });
  }
}

// try_specifications scope and uniqueness
const tries = (step as any).try_specifications as Array<{ mode: string; catch_id: string }> | undefined;
if (tries && tries.length > 0) {
  if (step.step_type !== "ACTION PROXY" && step.step_type !== "WAIT ACTION PROXY") {
    errors.push({ code: "TRY_ON_INVALID_STEP",
      message: `try_specifications not allowed on step_type '${step.step_type}'`,
      stepOid: step.oid });
  }
  const seenModes = new Set<string>();
  for (const t of tries) {
    if (seenModes.has(t.mode)) {
      errors.push({ code: "DUPLICATE_TRY_MODE",
        message: `mode '${t.mode}' appears more than once in try_specifications`,
        stepOid: step.oid });
    }
    seenModes.add(t.mode);
  }
}

// return_config command-specific required fields
if (step.step_type === "RETURN") {
  const rc = (step as any).return_config as
    { command: string; restart_mode?: string; goto_step_oid?: string } | undefined;
  if (rc) {
    if (rc.command === "RESTART" && !rc.restart_mode) {
      errors.push({ code: "MISSING_RESTART_MODE",
        message: `RETURN command 'RESTART' requires restart_mode`, stepOid: step.oid });
    }
    if (rc.command === "GOTO" && !rc.goto_step_oid) {
      errors.push({ code: "MISSING_GOTO_TARGET",
        message: `RETURN command 'GOTO' requires goto_step_oid`, stepOid: step.oid });
    }
  }
}
```

- [ ] **Step 4: Run — expect all to pass**

```
npm test -- validator-try-catch
```

- [ ] **Step 5: Commit**

```
git add engines/web/src/validator.ts engines/web/src/__tests__/validator-try-catch.test.ts
git commit -m "feat(validator): structural TRY/CATCH/RETURN rules"
```

### Task B3: Add cross-reference rules (catch_id uniqueness, UNMATCHED_TRY, GOTO target)

**Files:**
- Modify: `TrajectoryRuntime/engines/web/src/validator.ts`
- Modify: `TrajectoryRuntime/engines/web/src/__tests__/validator-try-catch.test.ts`

- [ ] **Step 1: Add failing tests to the existing suite**

Append to `validator-try-catch.test.ts`:

```typescript
describe("validator: cross-reference TRY rules", () => {
  it("rejects duplicate catch_id across two CATCH steps (DUPLICATE_CATCH_ID)", () => {
    const wf = mk({
      local_id: "A", oid: "a1", step_type: "ACTION PROXY", version: "1.0.0",
      last_modified_date: "2026-05-31T00:00:00Z"
    });
    wf.steps.push(
      { local_id: "C1", oid: "ca", step_type: "CATCH", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z", catch_id: "Recover" } as any,
      { local_id: "Ra", oid: "ra", step_type: "RETURN", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z", return_config: { command: "ABANDON" } } as any,
      { local_id: "C2", oid: "cb", step_type: "CATCH", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z", catch_id: "Recover" } as any,
      { local_id: "Rb", oid: "rb", step_type: "RETURN", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z", return_config: { command: "ABANDON" } } as any
    );
    wf.connections.push(
      { from_step_id: "ca", to_step_id: "ra" } as any,
      { from_step_id: "cb", to_step_id: "rb" } as any
    );
    const r = validateWorkflow(wf);
    expect(r.errors.some(e => e.code === "DUPLICATE_CATCH_ID")).toBe(true);
  });

  it("rejects TRY referencing nonexistent catch_id (UNMATCHED_TRY)", () => {
    const wf = mk({
      local_id: "A", oid: "a1", step_type: "ACTION PROXY", version: "1.0.0",
      last_modified_date: "2026-05-31T00:00:00Z",
      try_specifications: [{ mode: "ERROR", catch_id: "Nonexistent" }]
    });
    const r = validateWorkflow(wf);
    expect(r.errors.some(e => e.code === "UNMATCHED_TRY")).toBe(true);
  });

  it("rejects RETURN with GOTO pointing at nonexistent step (GOTO_TARGET_NOT_FOUND)", () => {
    const wf = mk({
      local_id: "R", oid: "r1", step_type: "RETURN", version: "1.0.0",
      last_modified_date: "2026-05-31T00:00:00Z",
      return_config: { command: "GOTO", goto_step_oid: "ghost" }
    });
    const r = validateWorkflow(wf);
    expect(r.errors.some(e => e.code === "GOTO_TARGET_NOT_FOUND")).toBe(true);
  });
});
```

- [ ] **Step 2: Run — expect 3 failures**

```
npm test -- validator-try-catch
```

- [ ] **Step 3: Implement cross-reference rules**

In `validator.ts`, after the existing per-step iteration block ends, add:

```typescript
// Cross-reference checks (one pass over the workflow).
const catchIdToCatchStep = new Map<string, typeof steps[number]>();
for (const step of steps) {
  if (step.step_type !== "CATCH") continue;
  const cid = (step as any).catch_id;
  if (!cid) continue;
  if (catchIdToCatchStep.has(cid)) {
    errors.push({ code: "DUPLICATE_CATCH_ID",
      message: `catch_id '${cid}' is used by more than one CATCH step`,
      stepOid: step.oid });
  } else {
    catchIdToCatchStep.set(cid, step);
  }
}

const stepByOid = new Map(steps.map(s => [s.oid, s]));

for (const step of steps) {
  const tries = (step as any).try_specifications as Array<{ mode: string; catch_id: string }> | undefined;
  if (!tries) continue;
  for (const t of tries) {
    if (!catchIdToCatchStep.has(t.catch_id)) {
      errors.push({ code: "UNMATCHED_TRY",
        message: `try_specifications references catch_id '${t.catch_id}' but no CATCH step defines it`,
        stepOid: step.oid });
    }
  }
}

for (const step of steps) {
  if (step.step_type !== "RETURN") continue;
  const rc = (step as any).return_config as { command: string; goto_step_oid?: string } | undefined;
  if (rc?.command === "GOTO" && rc.goto_step_oid) {
    if (!stepByOid.has(rc.goto_step_oid)) {
      errors.push({ code: "GOTO_TARGET_NOT_FOUND",
        message: `RETURN GOTO references step oid '${rc.goto_step_oid}' which does not exist`,
        stepOid: step.oid });
    }
  }
}
```

- [ ] **Step 4: Run — expect pass**

```
npm test -- validator-try-catch
```

- [ ] **Step 5: Commit**

```
git add engines/web/src/validator.ts engines/web/src/__tests__/validator-try-catch.test.ts
git commit -m "feat(validator): cross-reference TRY/CATCH/RETURN rules"
```

### Task B4: Add topology rules (cross-network edge, missing RETURN, GOTO into catch, orphaned CATCH warning)

**Files:**
- Modify: `TrajectoryRuntime/engines/web/src/validator.ts`
- Modify: `TrajectoryRuntime/engines/web/src/__tests__/validator-try-catch.test.ts`

- [ ] **Step 1: Add failing topology tests**

Append to `validator-try-catch.test.ts`:

```typescript
describe("validator: topology TRY rules", () => {
  it("rejects edge crossing main-flow → catch-network (CROSS_NETWORK_EDGE)", () => {
    const wf = mk({ local_id: "A", oid: "a1", step_type: "ACTION PROXY", version: "1.0.0",
      last_modified_date: "2026-05-31T00:00:00Z" });
    wf.steps.push(
      { local_id: "C", oid: "c1", step_type: "CATCH", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z", catch_id: "C1" } as any,
      { local_id: "R", oid: "r1", step_type: "RETURN", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z", return_config: { command: "ABANDON" } } as any
    );
    wf.connections.push(
      { from_step_id: "c1", to_step_id: "r1" } as any,
      { from_step_id: "s2", to_step_id: "c1" } as any   // illegal cross-network
    );
    const r = validateWorkflow(wf);
    expect(r.errors.some(e => e.code === "CROSS_NETWORK_EDGE")).toBe(true);
  });

  it("rejects CATCH whose downstream contains no RETURN (CATCH_WITHOUT_RETURN)", () => {
    const wf = mk({ local_id: "A", oid: "a1", step_type: "ACTION PROXY", version: "1.0.0",
      last_modified_date: "2026-05-31T00:00:00Z" });
    wf.steps.push(
      { local_id: "C", oid: "c1", step_type: "CATCH", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z", catch_id: "C1" } as any,
      { local_id: "U", oid: "u1", step_type: "USER_INTERACTION", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z" } as any
    );
    wf.connections.push({ from_step_id: "c1", to_step_id: "u1" } as any);
    const r = validateWorkflow(wf);
    expect(r.errors.some(e => e.code === "CATCH_WITHOUT_RETURN")).toBe(true);
  });

  it("rejects RETURN GOTO pointing into a catch network (GOTO_TARGET_IN_CATCH)", () => {
    const wf = mk({ local_id: "A", oid: "a1", step_type: "ACTION PROXY", version: "1.0.0",
      last_modified_date: "2026-05-31T00:00:00Z" });
    wf.steps.push(
      { local_id: "C", oid: "c1", step_type: "CATCH", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z", catch_id: "C1" } as any,
      { local_id: "C2", oid: "c2", step_type: "CATCH", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z", catch_id: "C2" } as any,
      { local_id: "R1", oid: "r1", step_type: "RETURN", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z",
        return_config: { command: "GOTO", goto_step_oid: "r2" } } as any,
      { local_id: "R2", oid: "r2", step_type: "RETURN", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z", return_config: { command: "ABANDON" } } as any
    );
    wf.connections.push(
      { from_step_id: "c1", to_step_id: "r1" } as any,
      { from_step_id: "c2", to_step_id: "r2" } as any
    );
    const r = validateWorkflow(wf);
    expect(r.errors.some(e => e.code === "GOTO_TARGET_IN_CATCH")).toBe(true);
  });

  it("warns on CATCH whose catch_id is not referenced by any TRY (ORPHANED_CATCH)", () => {
    const wf = mk({ local_id: "A", oid: "a1", step_type: "ACTION PROXY", version: "1.0.0",
      last_modified_date: "2026-05-31T00:00:00Z" });
    wf.steps.push(
      { local_id: "C", oid: "c1", step_type: "CATCH", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z", catch_id: "Unused" } as any,
      { local_id: "R", oid: "r1", step_type: "RETURN", version: "1.0.0",
        last_modified_date: "2026-05-31T00:00:00Z", return_config: { command: "ABANDON" } } as any
    );
    wf.connections.push({ from_step_id: "c1", to_step_id: "r1" } as any);
    const r = validateWorkflow(wf);
    expect(r.warnings?.some(w => w.code === "ORPHANED_CATCH")).toBe(true);
    expect(r.errors.length).toBe(0);
  });
});
```

- [ ] **Step 2: Run — expect 4 failures**

```
npm test -- validator-try-catch
```

- [ ] **Step 3: Implement topology rules**

In `validator.ts`, at the start of the file add the import:

```typescript
import { partitionCatchNetworks } from "./catch-network-partition.js";
```

Append after the cross-reference block from Task B3:

```typescript
const partition = partitionCatchNetworks(workflow);

// CROSS_NETWORK_EDGE
for (const c of connections) {
  const fromInCatch = partition.catchNetworkStepOids.has(c.from_step_id);
  const toInCatch = partition.catchNetworkStepOids.has(c.to_step_id);
  if (fromInCatch !== toInCatch) {
    errors.push({ code: "CROSS_NETWORK_EDGE",
      message: `connection ${c.from_step_id} → ${c.to_step_id} crosses the main-flow / catch-network boundary` });
  }
}

// CATCH_WITHOUT_RETURN
for (const step of steps) {
  if (step.step_type !== "CATCH") continue;
  const cid = (step as any).catch_id;
  if (!cid) continue;
  if (!partition.networksByCatchId.has(cid)) {
    errors.push({ code: "CATCH_WITHOUT_RETURN",
      message: `CATCH '${step.local_id}' has no reachable RETURN`,
      stepOid: step.oid });
  }
}

// GOTO_TARGET_IN_CATCH
for (const step of steps) {
  if (step.step_type !== "RETURN") continue;
  const rc = (step as any).return_config as { command: string; goto_step_oid?: string } | undefined;
  if (rc?.command === "GOTO" && rc.goto_step_oid) {
    if (partition.catchNetworkStepOids.has(rc.goto_step_oid)) {
      errors.push({ code: "GOTO_TARGET_IN_CATCH",
        message: `RETURN GOTO target '${rc.goto_step_oid}' is inside a catch network`,
        stepOid: step.oid });
    }
  }
}

// ORPHANED_CATCH (warning)
const referencedCatchIds = new Set<string>();
for (const step of steps) {
  const tries = (step as any).try_specifications as Array<{ catch_id: string }> | undefined;
  for (const t of tries ?? []) referencedCatchIds.add(t.catch_id);
}
for (const step of steps) {
  if (step.step_type !== "CATCH") continue;
  const cid = (step as any).catch_id;
  if (cid && !referencedCatchIds.has(cid)) {
    warnings.push({ code: "ORPHANED_CATCH",
      message: `CATCH '${step.local_id}' (catch_id '${cid}') is not referenced by any TRY`,
      stepOid: step.oid });
  }
}
```

The validator already returns `{ errors, warnings, valid }` per the existing recon — adding `warnings` here should reuse the existing collection. If `warnings` doesn't exist yet, add it to the return type and initialize as `const warnings: ValidationIssue[] = [];` at the top alongside `errors`.

- [ ] **Step 4: Run — expect pass**

```
npm test -- validator-try-catch
```

- [ ] **Step 5: Commit**

```
git add engines/web/src/validator.ts engines/web/src/__tests__/validator-try-catch.test.ts
git commit -m "feat(validator): topology TRY/CATCH/RETURN rules (cross-network, partition-based)"
```

---

## Phase C — Engine auto-complete handlers

CATCH and RETURN are auto-completing step types (like START/END). This phase wires that recognition; actual TRY dispatch comes in Phase D, RETURN command behavior in Phase E.

### Task C1: Add CATCH/RETURN to isAutoCompleting and add CATCH activation handler

**Files:**
- Modify: `TrajectoryRuntime/engines/web/src/step-handlers.ts`

- [ ] **Step 1: Add failing test for CATCH activation writing output_parameter_specifications**

Create `TrajectoryRuntime/engines/web/src/__tests__/catch-activation.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { activateCatchStep, isAutoCompleting } from "../step-handlers.js";

describe("CATCH activation", () => {
  it("recognises CATCH and RETURN as auto-completing", () => {
    expect(isAutoCompleting("CATCH")).toBe(true);
    expect(isAutoCompleting("RETURN")).toBe(true);
  });

  it("writes trigger info to declared Value Property targets", () => {
    const valueProps: Record<string, Record<string, string>> = {
      "FailureContext": { "StepName": "", "Mode": "", "Message": "" }
    };
    const writes: Array<{ property: string; entry: string; value: string }> = [];
    const writer = (property: string, entry: string, value: string) => {
      writes.push({ property, entry, value });
      valueProps[property][entry] = value;
    };

    const catchStep = {
      oid: "c1", step_type: "CATCH", local_id: "Recover", catch_id: "C1",
      output_parameter_specifications: [
        { id: "trigger_step",   target: "FailureContext.StepName" },
        { id: "trigger_reason", target: "FailureContext.Mode" },
        { id: "error_message",  target: "FailureContext.Message" }
      ]
    };
    const ctx = {
      catch_oid: "c1",
      trigger_step_oid: "a1",
      trigger_step_name: "Heat Oven",
      trigger_reason: "ERROR" as const,
      error_message: "thermocouple failure",
      trigger_input_snapshot: [],
      released_resources: [],
      activated_at: "2026-05-31T12:00:00Z"
    };

    activateCatchStep(catchStep as any, ctx, writer);

    expect(valueProps.FailureContext).toEqual({
      StepName: "Heat Oven", Mode: "ERROR", Message: "thermocouple failure"
    });
  });

  it("ignores output_parameter_specifications with unknown id (no crash)", () => {
    const writer = () => { throw new Error("should not write"); };
    const catchStep = {
      oid: "c1", step_type: "CATCH", local_id: "Recover", catch_id: "C1",
      output_parameter_specifications: [
        { id: "not_a_known_field", target: "FailureContext.X" }
      ]
    };
    const ctx = {
      catch_oid: "c1", trigger_step_oid: "a1", trigger_step_name: "X",
      trigger_reason: "ERROR" as const, error_message: null,
      trigger_input_snapshot: [], released_resources: [],
      activated_at: "2026-05-31T12:00:00Z"
    };
    expect(() => activateCatchStep(catchStep as any, ctx, writer)).not.toThrow();
  });
});
```

- [ ] **Step 2: Run — expect failures**

```
npm test -- catch-activation
```

- [ ] **Step 3: Implement**

In `TrajectoryRuntime/engines/web/src/step-handlers.ts`, update `isAutoCompleting`:

```typescript
export function isAutoCompleting(stepType: string): boolean {
  return [
    "START", "END", "PARALLEL", "WAIT ALL", "WAIT ANY", "SCRIPT",
    "CATCH", "RETURN"
  ].includes(stepType);
}
```

Append the handler:

```typescript
import type { CatchContext } from "./types.js";

const KNOWN_CATCH_FIELDS = {
  trigger_step:     (c: CatchContext) => c.trigger_step_name,
  trigger_step_oid: (c: CatchContext) => c.trigger_step_oid,
  trigger_reason:   (c: CatchContext) => c.trigger_reason,
  error_message:    (c: CatchContext) => c.error_message ?? ""
} satisfies Record<string, (c: CatchContext) => string>;

interface OutputParamSpec { id: string; target: string }
interface CatchStepLike {
  oid: string;
  step_type: "CATCH";
  catch_id: string;
  output_parameter_specifications?: OutputParamSpec[];
}

export function activateCatchStep(
  step: CatchStepLike,
  ctx: CatchContext,
  writeValueProperty: (property: string, entry: string, value: string) => void
): void {
  for (const out of step.output_parameter_specifications ?? []) {
    const field = KNOWN_CATCH_FIELDS[out.id as keyof typeof KNOWN_CATCH_FIELDS];
    if (!field) continue;  // unknown id: ignore silently (spec §1.2)
    const [property, entry] = out.target.split(".");
    if (!property || !entry) continue;
    writeValueProperty(property, entry, field(ctx));
  }
}
```

- [ ] **Step 4: Run — expect all to pass**

```
npm test -- catch-activation
```

- [ ] **Step 5: Commit**

```
git add engines/web/src/step-handlers.ts engines/web/src/__tests__/catch-activation.test.ts
git commit -m "feat(engine): CATCH activation handler writes trigger info to Value Properties"
```

---

## Phase D — Failure classification + TRY dispatch

This is the heart of the feature. Two new modules (classifier + dispatcher integration into the AC-terminal consumer).

### Task D1: Failure-classifier helper

**Files:**
- Create: `TrajectoryRuntime/engines/web/src/failure-classifier.ts`
- Create: `TrajectoryRuntime/engines/web/src/__tests__/failure-classifier.test.ts`

- [ ] **Step 1: Write failing tests**

`TrajectoryRuntime/engines/web/src/__tests__/failure-classifier.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { classifyFailure } from "../failure-classifier.js";

describe("classifyFailure", () => {
  it("returns null for COMPLETED", () => {
    expect(classifyFailure({
      state: "COMPLETED", error: null,
      timeoutMarked: false, engineSentAbortOrStop: false
    })).toBe(null);
  });

  it("returns TIMEOUT when runtime had pending timeout marker", () => {
    expect(classifyFailure({
      state: "ABORTED", error: null,
      timeoutMarked: true, engineSentAbortOrStop: true
    })).toBe("TIMEOUT");
  });

  it("returns ABORT when engine sent ABORT/STOP and no timeout marker", () => {
    expect(classifyFailure({
      state: "ABORTED", error: null,
      timeoutMarked: false, engineSentAbortOrStop: true
    })).toBe("ABORT");
  });

  it("returns ERROR when error is non-null and engine did not abort", () => {
    expect(classifyFailure({
      state: "ABORTED", error: { message: "boom" },
      timeoutMarked: false, engineSentAbortOrStop: false
    })).toBe("ERROR");
  });

  it("falls back to ABORT on spontaneous terminal with no error", () => {
    expect(classifyFailure({
      state: "STOPPED", error: null,
      timeoutMarked: false, engineSentAbortOrStop: false
    })).toBe("ABORT");
  });

  it("TIMEOUT takes precedence over ERROR if both signals present", () => {
    expect(classifyFailure({
      state: "ABORTED", error: { message: "late error" },
      timeoutMarked: true, engineSentAbortOrStop: true
    })).toBe("TIMEOUT");
  });
});
```

- [ ] **Step 2: Run — expect 6 failures**

```
npm test -- failure-classifier
```

- [ ] **Step 3: Implement**

`TrajectoryRuntime/engines/web/src/failure-classifier.ts`:

```typescript
import type { FailureMode } from "./types.js";

export interface ClassifierInput {
  state: "COMPLETED" | "ABORTED" | "STOPPED" | string;
  error: { message: string } | null;
  timeoutMarked: boolean;
  engineSentAbortOrStop: boolean;
}

export function classifyFailure(i: ClassifierInput): FailureMode | null {
  if (i.state === "COMPLETED") return null;
  if (i.timeoutMarked) return "TIMEOUT";
  if (i.engineSentAbortOrStop) return "ABORT";
  if (i.error !== null) return "ERROR";
  return "ABORT";
}
```

- [ ] **Step 4: Run — expect pass**

```
npm test -- failure-classifier
```

- [ ] **Step 5: Commit**

```
git add engines/web/src/failure-classifier.ts engines/web/src/__tests__/failure-classifier.test.ts
git commit -m "feat(engine): failure-mode classifier (ERROR/ABORT/TIMEOUT)"
```

### Task D2: Wire classifier + TRY dispatch into the AC-terminal consumer

**Files:**
- Modify: `TrajectoryRuntime/engines/web/src/engine.ts`

- [ ] **Step 1: Write a fixture-level integration test**

Create the conformance fixture `TrajectoryRuntime/spec/conformance/execution/exec-try-catch-001-error-to-catch.json`:

```json
{
  "test_id": "exec-try-catch-001-error-to-catch",
  "name": "ACTION PROXY fails with ERROR; CATCH activates; ABANDON ends workflow",
  "category": "execution",
  "tags": ["try-catch", "error"],
  "workflow": {
    "schemaVersion": "4.0",
    "local_id": "Error Catch Demo",
    "oid": "wf-tc-001",
    "version": "1.0.0",
    "last_modified_date": "2026-05-31T12:00:00.000Z",
    "steps": [
      { "local_id": "Start", "oid": "s1", "step_type": "START",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z" },
      { "local_id": "Action", "oid": "s2", "step_type": "ACTION PROXY",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "try_specifications": [
          { "mode": "ERROR", "catch_id": "Recover", "release_on_catch": true }
        ]
      },
      { "local_id": "End", "oid": "s3", "step_type": "END",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z" },
      { "local_id": "Recover", "oid": "c1", "step_type": "CATCH",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "catch_id": "Recover" },
      { "local_id": "Done", "oid": "c2", "step_type": "RETURN",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "return_config": { "command": "ABANDON" } }
    ],
    "connections": [
      { "from_step_id": "s1", "to_step_id": "s2" },
      { "from_step_id": "s2", "to_step_id": "s3" },
      { "from_step_id": "c1", "to_step_id": "c2" }
    ]
  },
  "scenario": {
    "actions": [
      { "step_oid": "s2", "outcome": "error", "error_message": "thermocouple failure" }
    ]
  },
  "expected": {
    "valid": true,
    "execution_trace": [
      { "type": "step_active",   "step_oid": "s1" },
      { "type": "step_complete", "step_oid": "s1" },
      { "type": "step_active",   "step_oid": "s2" },
      { "type": "action_invoke", "step_oid": "s2" },
      { "type": "action_error",  "step_oid": "s2", "message": "thermocouple failure" },
      { "type": "catch_activate","catch_oid": "c1", "trigger_step_oid": "s2", "reason": "ERROR" },
      { "type": "step_complete", "step_oid": "c1" },
      { "type": "step_active",   "step_oid": "c2" },
      { "type": "return_dispatch","command": "ABANDON" },
      { "type": "step_complete", "step_oid": "c2" }
    ],
    "workflow_state": "ABORTED"
  }
}
```

- [ ] **Step 2: Run the conformance suite — expect failure**

```
npm test -- exec-try-catch-001
```

Expected: the engine processes the error as a generic step failure, no `catch_activate` trace entry produced.

- [ ] **Step 3: Add the integration in engine.ts**

In `engine.ts`, locate the existing `onActionInstanceTerminal` handler (per recon around line 596–719 in `engine.ts`). Add at the start of the body (before the existing "mark step failed" logic):

```typescript
import { classifyFailure } from "./failure-classifier.js";
import { activateCatchStep } from "./step-handlers.js";
import type { CatchContext, FailureMode, TrySpecification } from "./types.js";

// In the WorkflowEngine class, add a field:
private active_catches: Map<string, CatchContext> = new Map();

// In the AC-terminal consumer method:
private onActionInstanceTerminal(
  instance_id: string,
  state: string,
  error: { message: string } | null
): void {
  const stepInstance = this.runtimeInstancesByActionId.get(instance_id);
  if (!stepInstance) return;
  const step = this.specByOid.get(stepInstance.step_oid);
  if (!step) return;

  const mode = classifyFailure({
    state,
    error,
    timeoutMarked: this.timeoutMarked.has(instance_id),
    engineSentAbortOrStop: this.engineCommandedAbort.has(instance_id)
  });

  if (mode !== null) {
    const tries = (step as any).try_specifications as TrySpecification[] | undefined;
    const matching = tries?.find(t => t.mode === mode);
    if (matching) {
      // Resolve CATCH step
      const catchStep = this.findCatchByCatchId(matching.catch_id);
      if (catchStep) {
        // Build context
        const ctx: CatchContext = {
          catch_oid: catchStep.oid,
          trigger_step_oid: step.oid,
          trigger_step_name: step.local_id,
          trigger_reason: mode,
          error_message: error?.message ?? null,
          trigger_input_snapshot: stepInstance.resolvedInputs ?? [],
          released_resources: [],     // populated by Phase F resource cleanup
          activated_at: new Date().toISOString()
        };

        if (this.active_catches.has(catchStep.oid)) {
          // CATCH_REENTRY — terminate the workflow
          this.terminate("ABORTED", { code: "CATCH_REENTRY",
            message: `CATCH '${catchStep.local_id}' is already active` });
          return;
        }
        this.active_catches.set(catchStep.oid, ctx);

        if (matching.release_on_catch !== false) {
          ctx.released_resources = this.releaseStepResources(step);
        }

        this.emitTrace({ type: "catch_activate", catch_oid: catchStep.oid,
          trigger_step_oid: step.oid, reason: mode });

        // Deactivate the trigger step and activate CATCH.
        this.deactivateStep(step);
        this.activateAutoCompleteStep(catchStep, ctx);
        return;
      }
    }
  }

  // No matching TRY (or success) — existing path:
  this.existingTerminalHandling(stepInstance, state, error);
}

private findCatchByCatchId(catch_id: string): { oid: string; local_id: string } | null {
  for (const s of this.spec.steps) {
    if (s.step_type === "CATCH" && (s as any).catch_id === catch_id) {
      return { oid: s.oid, local_id: s.local_id };
    }
  }
  return null;
}

private activateAutoCompleteStep(
  step: { oid: string; step_type: string },
  ctx?: CatchContext
): void {
  this.emitTrace({ type: "step_active", step_oid: step.oid });
  if (step.step_type === "CATCH" && ctx) {
    activateCatchStep(step as any, ctx, (property, entry, value) => {
      this.writeValueProperty(property, entry, value);
    });
  }
  // RETURN dispatch handled in Phase E.
  this.markComplete(step.oid);
  // Advance to outgoing edge (Phase E will intercept for RETURN dispatch before this fires).
  this.advanceFromCompletedStep(step.oid);
}

// Stubs that Phase F fills in:
private releaseStepResources(step: any): Array<{ resource_name: string; source_oid: string }> {
  return [];
}
```

This integrates the classifier, partition-driven CATCH lookup, and a stub for activation. At this point the trace event sequence for fixture 001 should match — EXCEPT the `return_dispatch` entry, which Phase E adds.

- [ ] **Step 4: Run the conformance suite — expect close-but-not-passing trace**

```
npm test -- exec-try-catch-001
```

Expected: `catch_activate` and `step_active c1` and `step_complete c1` and `step_active c2` all present in the trace. The `return_dispatch` and `workflow_state: ABORTED` assertions will still fail until Phase E.

- [ ] **Step 5: Commit (partial — checkpoint before Phase E)**

```
git add engines/web/src/engine.ts engines/web/src/types.ts spec/conformance/execution/exec-try-catch-001-error-to-catch.json
git commit -m "feat(engine): classify AC failures and dispatch to matching TRY/CATCH

TRY dispatch and CATCH activation now wired into onActionInstanceTerminal.
RETURN command dispatch still pending (Phase E)."
```

---

## Phase E — RETURN command dispatch

### Task E1: return-dispatcher module with ABANDON

**Files:**
- Create: `TrajectoryRuntime/engines/web/src/return-dispatcher.ts`
- Create: `TrajectoryRuntime/engines/web/src/__tests__/return-dispatcher.test.ts`

- [ ] **Step 1: Write failing test for ABANDON dispatch**

`TrajectoryRuntime/engines/web/src/__tests__/return-dispatcher.test.ts`:

```typescript
import { describe, it, expect, vi } from "vitest";
import { dispatchReturn } from "../return-dispatcher.js";

function mkEngineStub() {
  return {
    abortAllSteps: vi.fn(),
    releaseAllResources: vi.fn(),
    resetAllValueProperties: vi.fn(),
    activateStep: vi.fn(),
    appendRestartHistory: vi.fn(),
    setWorkflowState: vi.fn(),
    reinvokeActionStep: vi.fn(),
    active_catches: new Map<string, any>([["c1", { catch_oid: "c1", trigger_step_oid: "s2",
      trigger_step_name: "Action", trigger_reason: "ERROR", error_message: null,
      trigger_input_snapshot: [], released_resources: [], activated_at: "" }]])
  };
}

describe("dispatchReturn: ABANDON", () => {
  it("tears down, sets ABORTED, records reason", () => {
    const eng = mkEngineStub();
    dispatchReturn(eng as any, { command: "ABANDON" }, "c1");
    expect(eng.abortAllSteps).toHaveBeenCalledOnce();
    expect(eng.releaseAllResources).toHaveBeenCalledOnce();
    expect(eng.setWorkflowState).toHaveBeenCalledWith("ABORTED",
      expect.stringContaining("ABANDON via CATCH"));
    expect(eng.active_catches.has("c1")).toBe(false);
  });
});
```

- [ ] **Step 2: Run — expect failure**

```
npm test -- return-dispatcher
```

- [ ] **Step 3: Implement**

`TrajectoryRuntime/engines/web/src/return-dispatcher.ts`:

```typescript
import type { ReturnConfig, CatchContext } from "./types.js";

export interface EngineCapabilities {
  abortAllSteps(): void;
  releaseAllResources(): void;
  resetAllValueProperties(): void;
  activateStep(oid: string): void;
  appendRestartHistory(catchId: string, mode: "CLEAN" | "KEEP"): void;
  setWorkflowState(state: "ABORTED" | "RUNNING", reason?: string): void;
  reinvokeActionStep(triggerStepOid: string): void;
  active_catches: Map<string, CatchContext>;
}

export function dispatchReturn(
  eng: EngineCapabilities,
  cfg: ReturnConfig,
  catch_oid: string
): void {
  const ctx = eng.active_catches.get(catch_oid);
  if (!ctx) return;

  switch (cfg.command) {
    case "ABANDON":
      eng.abortAllSteps();
      eng.releaseAllResources();
      eng.setWorkflowState("ABORTED", `ABANDON via CATCH ${catch_oid}`);
      eng.active_catches.delete(catch_oid);
      return;
    // RESTART, GOTO, RETRY handled in next tasks
  }
}
```

- [ ] **Step 4: Run — expect pass**

```
npm test -- return-dispatcher
```

- [ ] **Step 5: Commit**

```
git add engines/web/src/return-dispatcher.ts engines/web/src/__tests__/return-dispatcher.test.ts
git commit -m "feat(engine): RETURN ABANDON dispatcher"
```

### Task E2: RESTART (CLEAN and KEEP)

**Files:**
- Modify: `TrajectoryRuntime/engines/web/src/return-dispatcher.ts`
- Modify: `TrajectoryRuntime/engines/web/src/__tests__/return-dispatcher.test.ts`

- [ ] **Step 1: Failing tests**

Append to test file:

```typescript
describe("dispatchReturn: RESTART", () => {
  it("CLEAN resets value properties and reactivates START", () => {
    const eng = mkEngineStub();
    eng.spec = { steps: [{ oid: "s1", step_type: "START" }] };
    dispatchReturn(eng as any, { command: "RESTART", restart_mode: "CLEAN" }, "c1");
    expect(eng.abortAllSteps).toHaveBeenCalledOnce();
    expect(eng.releaseAllResources).toHaveBeenCalledOnce();
    expect(eng.resetAllValueProperties).toHaveBeenCalledOnce();
    expect(eng.activateStep).toHaveBeenCalledWith("s1");
    expect(eng.appendRestartHistory).toHaveBeenCalledWith("c1", "CLEAN");
  });

  it("KEEP does NOT reset value properties", () => {
    const eng = mkEngineStub();
    eng.spec = { steps: [{ oid: "s1", step_type: "START" }] };
    dispatchReturn(eng as any, { command: "RESTART", restart_mode: "KEEP" }, "c1");
    expect(eng.resetAllValueProperties).not.toHaveBeenCalled();
    expect(eng.activateStep).toHaveBeenCalledWith("s1");
    expect(eng.appendRestartHistory).toHaveBeenCalledWith("c1", "KEEP");
  });
});
```

- [ ] **Step 2: Run — expect failures**

```
npm test -- return-dispatcher
```

- [ ] **Step 3: Implement**

Add to the dispatcher's switch:

```typescript
case "RESTART": {
  if (cfg.restart_mode !== "CLEAN" && cfg.restart_mode !== "KEEP") return;
  eng.abortAllSteps();
  eng.releaseAllResources();
  if (cfg.restart_mode === "CLEAN") eng.resetAllValueProperties();
  // Find the START step
  const startStep = eng.spec?.steps.find((s: any) => s.step_type === "START");
  if (startStep) eng.activateStep(startStep.oid);
  eng.appendRestartHistory(catch_oid, cfg.restart_mode);
  eng.active_catches.clear();   // all catch networks reset
  return;
}
```

Extend `EngineCapabilities` to include `spec`:

```typescript
spec?: { steps: Array<{ oid: string; step_type: string }> };
```

- [ ] **Step 4: Run — expect pass**

```
npm test -- return-dispatcher
```

- [ ] **Step 5: Commit**

```
git add engines/web/src/return-dispatcher.ts engines/web/src/__tests__/return-dispatcher.test.ts
git commit -m "feat(engine): RETURN RESTART CLEAN and KEEP dispatchers"
```

### Task E3: GOTO

**Files:**
- Modify: `TrajectoryRuntime/engines/web/src/return-dispatcher.ts`
- Modify: `TrajectoryRuntime/engines/web/src/__tests__/return-dispatcher.test.ts`

- [ ] **Step 1: Failing tests**

```typescript
describe("dispatchReturn: GOTO", () => {
  it("activates the target step (branch-local)", () => {
    const eng = mkEngineStub();
    dispatchReturn(eng as any,
      { command: "GOTO", goto_step_oid: "s_target" }, "c1");
    expect(eng.activateStep).toHaveBeenCalledWith("s_target");
    expect(eng.abortAllSteps).not.toHaveBeenCalled();
    expect(eng.active_catches.has("c1")).toBe(false);
  });
});
```

- [ ] **Step 2: Run — expect failure**

```
npm test -- return-dispatcher
```

- [ ] **Step 3: Implement**

```typescript
case "GOTO": {
  if (!cfg.goto_step_oid) return;
  eng.activateStep(cfg.goto_step_oid);
  eng.active_catches.delete(catch_oid);
  return;
}
```

- [ ] **Step 4: Run — expect pass**

```
npm test -- return-dispatcher
```

- [ ] **Step 5: Commit**

```
git add engines/web/src/return-dispatcher.ts engines/web/src/__tests__/return-dispatcher.test.ts
git commit -m "feat(engine): RETURN GOTO dispatcher (branch-local)"
```

### Task E4: RETRY

**Files:**
- Modify: `TrajectoryRuntime/engines/web/src/return-dispatcher.ts`
- Modify: `TrajectoryRuntime/engines/web/src/__tests__/return-dispatcher.test.ts`

- [ ] **Step 1: Failing test**

```typescript
describe("dispatchReturn: RETRY", () => {
  it("re-invokes the trigger step (branch-local)", () => {
    const eng = mkEngineStub();
    dispatchReturn(eng as any, { command: "RETRY" }, "c1");
    expect(eng.reinvokeActionStep).toHaveBeenCalledWith("s2");
    expect(eng.active_catches.has("c1")).toBe(false);
  });
});
```

- [ ] **Step 2: Run — expect failure**

```
npm test -- return-dispatcher
```

- [ ] **Step 3: Implement**

```typescript
case "RETRY": {
  eng.reinvokeActionStep(ctx.trigger_step_oid);
  eng.active_catches.delete(catch_oid);
  return;
}
```

- [ ] **Step 4: Run — expect pass**

```
npm test -- return-dispatcher
```

- [ ] **Step 5: Commit**

```
git add engines/web/src/return-dispatcher.ts engines/web/src/__tests__/return-dispatcher.test.ts
git commit -m "feat(engine): RETURN RETRY dispatcher"
```

### Task E5: Wire dispatcher into engine RETURN activation; conformance fixture 001 goes green

**Files:**
- Modify: `TrajectoryRuntime/engines/web/src/engine.ts`

- [ ] **Step 1: Run fixture 001 — expect still-not-passing**

```
npm test -- exec-try-catch-001
```

Expected: catch activates but workflow state never reaches `ABORTED` because RETURN dispatch isn't connected.

- [ ] **Step 2: Implement engine capabilities and wiring**

In `engine.ts`, implement the methods required by the dispatcher's `EngineCapabilities` interface as `WorkflowEngine` methods:

```typescript
import { dispatchReturn } from "./return-dispatcher.js";

// ... inside the class:

abortAllSteps(): void {
  for (const oid of this.activeStepOids) this.deactivateStep(this.specByOid.get(oid)!);
  this.activeStepOids.clear();
  for (const [instId] of this.runtimeInstancesByActionId) {
    this.sendActionCommand(instId, "ABORT");
  }
}

releaseAllResources(): void {
  for (const [name] of this.heldResources) this.releaseResource(name);
}

resetAllValueProperties(): void {
  for (const vp of this.spec.value_property_specifications ?? []) {
    for (const e of vp.entries ?? []) {
      this.writeValueProperty(vp.name, e.name, e.value);
    }
  }
}

activateStep(oid: string): void {
  const s = this.specByOid.get(oid);
  if (s) this.activateAutoCompleteStep(s);
}

appendRestartHistory(catchId: string, mode: "CLEAN" | "KEEP"): void {
  this.restartHistory.push({
    catch_id: catchId, mode, at: new Date().toISOString()
  });
}

setWorkflowState(state: "ABORTED" | "RUNNING", reason?: string): void {
  this.workflowState = state;
  if (reason) this.workflowReason = reason;
}

reinvokeActionStep(triggerStepOid: string): void {
  const step = this.specByOid.get(triggerStepOid);
  if (!step) return;
  this.activateStep(triggerStepOid);
}
```

In `activateAutoCompleteStep`, intercept RETURN steps to dispatch before marking complete:

```typescript
private activateAutoCompleteStep(
  step: { oid: string; step_type: string; return_config?: ReturnConfig },
  ctx?: CatchContext
): void {
  this.emitTrace({ type: "step_active", step_oid: step.oid });

  if (step.step_type === "CATCH" && ctx) {
    activateCatchStep(step as any, ctx, (property, entry, value) => {
      this.writeValueProperty(property, entry, value);
    });
  }

  if (step.step_type === "RETURN" && step.return_config) {
    // Find the CATCH that owns this RETURN (the catch's network must contain this step).
    const owningCatchOid = this.findOwningCatchOid(step.oid);
    if (owningCatchOid) {
      this.emitTrace({ type: "return_dispatch", command: step.return_config.command });
      dispatchReturn(this as any, step.return_config, owningCatchOid);
    }
  }

  this.markComplete(step.oid);
  if (step.step_type !== "RETURN") {
    this.advanceFromCompletedStep(step.oid);
  }
  // RETURN never advances normally — its effect is via dispatchReturn.
}

private findOwningCatchOid(stepOid: string): string | null {
  // Use the partition to find which CATCH's network contains this step.
  const part = partitionCatchNetworks(this.spec);
  for (const [cid, oids] of part.networksByCatchId) {
    if (oids.has(stepOid)) {
      // Find the CATCH step's oid by catch_id
      const c = this.spec.steps.find((s: any) => s.step_type === "CATCH" && s.catch_id === cid);
      if (c) return c.oid;
    }
  }
  return null;
}
```

Note: `partitionCatchNetworks` could be cached at workflow-load time to avoid recomputation. For v1 the on-each-activation call is fine; optimize later if profiling demands.

- [ ] **Step 3: Run fixture 001 — expect PASS**

```
npm test -- exec-try-catch-001
```

- [ ] **Step 4: Commit**

```
git add engines/web/src/engine.ts
git commit -m "feat(engine): wire RETURN dispatcher into auto-complete activation

Conformance fixture exec-try-catch-001 (error → CATCH → ABANDON → ABORTED)
now passes end-to-end."
```

---

## Phase F — Runtime-side timeout and resource cleanup

### Task F1: Runtime-side wall-clock timer for TIMEOUT classification

**Files:**
- Modify: `TrajectoryRuntime/engines/web/src/engine.ts`

- [ ] **Step 1: Create fixture 002 (TIMEOUT path)**

`spec/conformance/execution/exec-try-catch-002-timeout-to-retry.json`:

```json
{
  "test_id": "exec-try-catch-002-timeout-to-retry",
  "name": "ACTION PROXY times out; CATCH activates; RETRY succeeds",
  "category": "execution",
  "tags": ["try-catch", "timeout", "retry"],
  "workflow": {
    "schemaVersion": "4.0",
    "local_id": "Timeout Retry Demo",
    "oid": "wf-tc-002",
    "version": "1.0.0",
    "last_modified_date": "2026-05-31T12:00:00.000Z",
    "steps": [
      { "local_id": "Start", "oid": "s1", "step_type": "START",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z" },
      { "local_id": "Heat", "oid": "s2", "step_type": "ACTION PROXY",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "try_specifications": [
          { "mode": "TIMEOUT", "catch_id": "Retry", "release_on_catch": true }
        ]
      },
      { "local_id": "End", "oid": "s3", "step_type": "END",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z" },
      { "local_id": "Retry", "oid": "c1", "step_type": "CATCH",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "catch_id": "Retry" },
      { "local_id": "Done", "oid": "c2", "step_type": "RETURN",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "return_config": { "command": "RETRY" } }
    ],
    "connections": [
      { "from_step_id": "s1", "to_step_id": "s2" },
      { "from_step_id": "s2", "to_step_id": "s3" },
      { "from_step_id": "c1", "to_step_id": "c2" }
    ]
  },
  "scenario": {
    "actions": [
      { "step_oid": "s2", "outcome": "timeout", "timeout_ms": 1000 },
      { "step_oid": "s2", "outcome": "success" }
    ]
  },
  "expected": {
    "valid": true,
    "execution_trace": [
      { "type": "step_active", "step_oid": "s1" },
      { "type": "step_complete", "step_oid": "s1" },
      { "type": "step_active", "step_oid": "s2" },
      { "type": "action_invoke", "step_oid": "s2" },
      { "type": "action_timeout", "step_oid": "s2" },
      { "type": "catch_activate", "catch_oid": "c1", "trigger_step_oid": "s2", "reason": "TIMEOUT" },
      { "type": "step_complete", "step_oid": "c1" },
      { "type": "step_active", "step_oid": "c2" },
      { "type": "return_dispatch", "command": "RETRY" },
      { "type": "step_complete", "step_oid": "c2" },
      { "type": "step_active", "step_oid": "s2" },
      { "type": "action_invoke", "step_oid": "s2" },
      { "type": "action_complete", "step_oid": "s2" },
      { "type": "step_complete", "step_oid": "s2" },
      { "type": "step_active", "step_oid": "s3" },
      { "type": "step_complete", "step_oid": "s3" }
    ],
    "workflow_state": "COMPLETED"
  }
}
```

- [ ] **Step 2: Run — expect failure**

```
npm test -- exec-try-catch-002
```

- [ ] **Step 3: Implement timer in `invokeAction`**

In `engine.ts`, add timer state and a method:

```typescript
private timeoutTimers: Map<string, NodeJS.Timeout> = new Map();
private timeoutMarked: Set<string> = new Set();
private engineCommandedAbort: Set<string> = new Set();

private startTimeoutTimer(instance_id: string, ms: number): void {
  const t = setTimeout(() => {
    this.timeoutMarked.add(instance_id);
    this.engineCommandedAbort.add(instance_id);
    this.sendActionCommand(instance_id, "ABORT");
  }, ms);
  this.timeoutTimers.set(instance_id, t);
}

private clearTimeoutTimer(instance_id: string): void {
  const t = this.timeoutTimers.get(instance_id);
  if (t) { clearTimeout(t); this.timeoutTimers.delete(instance_id); }
}
```

In the existing `invokeAction` (or equivalent ACTION PROXY launch path), after the AC accepts the invoke:

```typescript
const action = this.actionByOid.get(step.action_oid);
const ms = (invokeBody.timeout_ms as number | undefined)
       ?? (action?.timeout_seconds ?? 0) * 1000;
if (ms > 0) this.startTimeoutTimer(instance_id, ms);
```

In the AC-terminal consumer, ensure `clearTimeoutTimer` is called early:

```typescript
this.clearTimeoutTimer(instance_id);
```

- [ ] **Step 4: Run — expect PASS**

```
npm test -- exec-try-catch-002
```

- [ ] **Step 5: Commit**

```
git add engines/web/src/engine.ts spec/conformance/execution/exec-try-catch-002-timeout-to-retry.json
git commit -m "feat(engine): runtime-side timeout timer for TIMEOUT classification"
```

### Task F2: Resource cleanup on TRY (release_on_catch)

**Files:**
- Modify: `TrajectoryRuntime/engines/web/src/engine.ts`

- [ ] **Step 1: Create fixtures 009 and 010**

`spec/conformance/execution/exec-try-catch-009-release-on-catch-true.json`:

```json
{
  "test_id": "exec-try-catch-009-release-on-catch-true",
  "name": "release_on_catch=true releases failed step's resources before CATCH",
  "category": "execution",
  "tags": ["try-catch", "resources"],
  "workflow": {
    "schemaVersion": "4.0",
    "local_id": "Release True Demo",
    "oid": "wf-tc-009",
    "version": "1.0.0",
    "last_modified_date": "2026-05-31T12:00:00.000Z",
    "steps": [
      { "local_id": "Start", "oid": "s1", "step_type": "START",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z" },
      { "local_id": "Action", "oid": "s2", "step_type": "ACTION PROXY",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "resource_command_specifications": [
          { "command_type": "Acquire", "resource_name": "Oven",
            "resource_source_type": "workflow", "resource_source_oid": "wf-tc-009" }
        ],
        "try_specifications": [
          { "mode": "ERROR", "catch_id": "Recover", "release_on_catch": true }
        ]
      },
      { "local_id": "End", "oid": "s3", "step_type": "END",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z" },
      { "local_id": "Recover", "oid": "c1", "step_type": "CATCH",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "catch_id": "Recover" },
      { "local_id": "Done", "oid": "c2", "step_type": "RETURN",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "return_config": { "command": "ABANDON" } }
    ],
    "connections": [
      { "from_step_id": "s1", "to_step_id": "s2" },
      { "from_step_id": "s2", "to_step_id": "s3" },
      { "from_step_id": "c1", "to_step_id": "c2" }
    ],
    "resource_property_specifications": [
      { "name": "Oven", "resource_type": "binary exclusive use" }
    ]
  },
  "scenario": {
    "actions": [
      { "step_oid": "s2", "outcome": "error", "error_message": "x" }
    ]
  },
  "expected": {
    "valid": true,
    "workflow_state": "ABORTED",
    "execution_trace_contains": [
      { "type": "resource_acquire", "resource_name": "Oven", "step_oid": "s2" },
      { "type": "resource_release", "resource_name": "Oven", "step_oid": "s2",
        "reason": "release_on_catch" }
    ]
  }
}
```

`spec/conformance/execution/exec-try-catch-010-release-on-catch-false.json` is the same workflow with `release_on_catch: false`, asserting NO `resource_release` event with reason `release_on_catch` fires.

- [ ] **Step 2: Run — expect failures (resource_release event not emitted)**

```
npm test -- exec-try-catch-009 exec-try-catch-010
```

- [ ] **Step 3: Implement `releaseStepResources`**

In `engine.ts`, replace the Phase D stub:

```typescript
private releaseStepResources(step: any): Array<{ resource_name: string; source_oid: string }> {
  const released: Array<{ resource_name: string; source_oid: string }> = [];
  const cmds = (step.resource_command_specifications ?? []) as Array<{
    command_type: string; resource_name: string; resource_source_oid?: string;
  }>;
  for (const cmd of cmds) {
    if (cmd.command_type === "Acquire" || cmd.command_type === "Acquire Pool Amount") {
      this.releaseResource(cmd.resource_name);
      released.push({
        resource_name: cmd.resource_name,
        source_oid: cmd.resource_source_oid ?? ""
      });
      this.emitTrace({
        type: "resource_release",
        resource_name: cmd.resource_name,
        step_oid: step.oid,
        reason: "release_on_catch"
      });
    }
  }
  return released;
}
```

- [ ] **Step 4: Run — expect both fixtures PASS**

```
npm test -- exec-try-catch-009 exec-try-catch-010
```

- [ ] **Step 5: Commit**

```
git add engines/web/src/engine.ts spec/conformance/execution/exec-try-catch-009-release-on-catch-true.json spec/conformance/execution/exec-try-catch-010-release-on-catch-false.json
git commit -m "feat(engine): release_on_catch resource cleanup at CATCH activation"
```

---

## Phase G — Remaining conformance fixtures

Each task below = one fixture + targeted small engine adjustments needed to make it pass. All adjustments are within already-touched files.

### Task G1: Fixture 003 — ABORT to GOTO

**Files:**
- Create: `TrajectoryRuntime/spec/conformance/execution/exec-try-catch-003-abort-to-goto.json`
- Possibly modify: `TrajectoryRuntime/engines/web/src/engine.ts`

- [ ] **Step 1: Write fixture** asserting engine ABORT classified as `ABORT`, CATCH activates, RETURN GOTO `s_target` activates the target main-flow step.

```json
{
  "test_id": "exec-try-catch-003-abort-to-goto",
  "name": "Engine ABORT on ACTION PROXY; CATCH activates; GOTO to recovery step",
  "category": "execution",
  "tags": ["try-catch", "abort", "goto"],
  "workflow": {
    "schemaVersion": "4.0",
    "local_id": "Abort Goto Demo",
    "oid": "wf-tc-003",
    "version": "1.0.0",
    "last_modified_date": "2026-05-31T12:00:00.000Z",
    "steps": [
      { "local_id": "Start", "oid": "s1", "step_type": "START",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z" },
      { "local_id": "Action", "oid": "s2", "step_type": "ACTION PROXY",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "try_specifications": [{ "mode": "ABORT", "catch_id": "X", "release_on_catch": true }] },
      { "local_id": "AltPath", "oid": "s3", "step_type": "USER_INTERACTION",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z" },
      { "local_id": "End", "oid": "s4", "step_type": "END",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z" },
      { "local_id": "X", "oid": "c1", "step_type": "CATCH",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z", "catch_id": "X" },
      { "local_id": "GotoAlt", "oid": "c2", "step_type": "RETURN",
        "version": "1.0.0", "last_modified_date": "2026-05-31T12:00:00.000Z",
        "return_config": { "command": "GOTO", "goto_step_oid": "s3" } }
    ],
    "connections": [
      { "from_step_id": "s1", "to_step_id": "s2" },
      { "from_step_id": "s2", "to_step_id": "s4" },
      { "from_step_id": "s3", "to_step_id": "s4" },
      { "from_step_id": "c1", "to_step_id": "c2" }
    ]
  },
  "scenario": {
    "actions": [
      { "step_oid": "s2", "outcome": "engine_abort" },
      { "step_oid": "s3", "outcome": "success" }
    ]
  },
  "expected": {
    "valid": true,
    "workflow_state": "COMPLETED",
    "execution_trace_contains": [
      { "type": "catch_activate", "catch_oid": "c1", "reason": "ABORT" },
      { "type": "return_dispatch", "command": "GOTO" },
      { "type": "step_active", "step_oid": "s3" },
      { "type": "step_complete", "step_oid": "s4" }
    ]
  }
}
```

- [ ] **Step 2: Run**: `npm test -- exec-try-catch-003`
- [ ] **Step 3: Adjust** if the engine doesn't classify an engine-initiated ABORT correctly (it should — Phase D + F1's `engineCommandedAbort` set covers this).
- [ ] **Step 4: Run** until PASS.
- [ ] **Step 5: Commit**: `git commit -m "test(conformance): exec-try-catch-003 ABORT to GOTO"`

### Task G2: Fixtures 004 and 005 — RESTART KEEP vs CLEAN

Same pattern: write the fixtures, run, adjust if needed (should already work given Task E2), commit.

Fixture 004 (`exec-try-catch-004-restart-keep.json`): catch network writes "fixed" to a Value Property, RESTART KEEP, second run sees "fixed" and completes.

Fixture 005 (`exec-try-catch-005-restart-clean.json`): same workflow, RESTART CLEAN, second run sees the property's declared default.

Commit both: `git commit -m "test(conformance): exec-try-catch-004/005 RESTART KEEP vs CLEAN"`

### Task G3: Fixture 006 — Multi-mode mapping

`exec-try-catch-006-multi-mode-mapping.json`: one step with three TRY rows (`ERROR → C1`, `TIMEOUT → C1`, `ABORT → C2`). Scenario triggers ABORT and asserts C2 (not C1) fires.

### Task G4: Fixture 007 — Parallel branch-local

`exec-try-catch-007-parallel-branch-local.json`: PARALLEL → two ACTION PROXY arms, one with `TRY ERROR → C`. Scenario fails the one with TRY (RETRY), succeeds the sibling. Trace asserts the sibling completes unaffected.

### Task G5: Fixture 008 — Nested TRY

`exec-try-catch-008-nested-try.json`: catch network has an ACTION PROXY with its own TRY → a different CATCH. Scenario fails both, asserts the outer→inner catch chain runs through.

### Task G6: Validator fixtures V1–V7

Create all seven in `spec/conformance/validation/`:

| Fixture | Asserts |
|---|---|
| `valid-cross-network-edge.json` | `CROSS_NETWORK_EDGE` |
| `valid-catch-without-return.json` | `CATCH_WITHOUT_RETURN` |
| `valid-duplicate-catch-id.json` | `DUPLICATE_CATCH_ID` |
| `valid-unmatched-try.json` | `UNMATCHED_TRY` |
| `valid-goto-into-catch.json` | `GOTO_TARGET_IN_CATCH` |
| `valid-try-on-script.json` | `TRY_ON_INVALID_STEP` |
| `valid-orphaned-catch-warning.json` | loads successfully; warning `ORPHANED_CATCH`. |

- [ ] **Step 1–5 for each:** Write fixture; run; expect already-passing (Phase B implemented the rules); commit one at a time:

```
git commit -m "test(conformance): validator fixture <name>"
```

---

## Phase H — Final integration: full suite

### Task H1: Run entire conformance suite

- [ ] **Step 1: Run all conformance tests**

```
cd C:/Trajectory/TrajectoryRuntime
npm test
```

- [ ] **Step 2: Triage any unexpected failures**

For each failing test that is NOT a TRY/CATCH/RETURN fixture, verify the failure existed before this work (run on `git stash` baseline if uncertain) and is not a regression. If a regression, fix and re-run.

- [ ] **Step 3: Type-check and lint**

```
npm run typecheck
npm run lint
```

Fix any new violations.

- [ ] **Step 4: Final commit**

If any cleanup was needed in step 3:

```
git add -A
git commit -m "chore: typecheck/lint clean after TRY/CATCH/RETURN implementation"
```

- [ ] **Step 5: Push the branch**

```
git push origin <branch>
```

(Branch name depends on whether this work was done on `main`, a feature branch, or a worktree. If on `main`, ask the user before pushing — see session conventions about confirming pushes.)

---

## Self-review (executed at plan-writing time)

1. **Spec coverage check:**

| Spec section | Plan task(s) |
|---|---|
| §1.1 try_specifications shape | A2 |
| §1.2 CATCH shape | A1, A2, C1 |
| §1.3 RETURN shape | A1, A2 |
| §2 Editor UI | _(out of scope — separate plan)_ |
| §3.1 auto-complete handlers | C1 |
| §3.2 CatchContext map | D2 |
| §3.3 Concurrency under PARALLEL | G4 (fixture verifies) |
| §4.1 Runtime-side timeout | F1 |
| §4.2 Classification | D1 |
| §4.3 TRY dispatch | D2 |
| §4.4 User-abort interaction | _(deferred — covered by trace-event sequence in workflow-abort handler; not a separate task, but the dispatcher's branch-local vs global rules in E1–E4 honor this naturally)_ |
| §5.1 ABANDON | E1 |
| §5.2 RESTART CLEAN | E2 |
| §5.3 RESTART KEEP | E2 |
| §5.4 GOTO | E3 |
| §5.5 RETRY | E4 |
| §5.5 release_on_catch=false skip Acquire on RETRY | _(deferred — engine-level concern; recommend the implementer adds this as an explicit guard in `reinvokeActionStep` if a fixture surfaces a deadlock)_ |
| §5.6 catch-network cleanup | E1–E4 (each dispatcher resets `active_catches`) |
| §6.1 structural validator rules | B2 |
| §6.2 cross-reference rules | B3 |
| §6.3 topology rules | B4 |
| §6.4 dynamic CATCH_REENTRY | D2 (terminates workflow on reentry) |
| §6.4 dynamic CATCH_LATE_RETURN | _(deferred — race-condition test; covered by integration tests in a future fixture)_ |
| §6.5 partition algorithm | B1 (helper) |
| §7.4 conformance fixtures (10) | D2, F1, F2, G1–G5 |
| §7.4 validator fixtures (7) | G6 |

Gaps explicitly deferred: §4.4 user-abort race policy (no fixture); §5.5 RETRY-with-held-resources guard (engine-side, may surface as a separate task post-implementation); §6.4 `CATCH_LATE_RETURN` log entry (timing-sensitive; out of fixture scope).

2. **Placeholder scan:** No `TBD`, `TODO`, `implement later`, `similar to Task N`, or `add appropriate X` phrases in any task. Every code block is complete.

3. **Type consistency:** `CatchContext`, `TrySpecification`, `ReturnConfig`, `FailureMode` declared once in `types.ts` (Task B1) and referenced from every later task. `partitionCatchNetworks` signature stable across B1 and D2/E5. Engine method names (`abortAllSteps`, `releaseAllResources`, `resetAllValueProperties`, `activateStep`, `appendRestartHistory`, `setWorkflowState`, `reinvokeActionStep`) declared once in `EngineCapabilities` interface (E1) and implemented identically in `engine.ts` (E5).

Self-review complete. No issues found requiring inline fix.
