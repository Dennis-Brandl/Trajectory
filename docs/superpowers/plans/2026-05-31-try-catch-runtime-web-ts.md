# TRY / CATCH / RETURN — Web (TypeScript) Runtime Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement TRY/CATCH/RETURN error-handling in the **TypeScript** workflow engine that ships on web (`engines/web/src`) and wire it through the production coordinator, so a failed/aborted/timed-out action routes to a CATCH network and executes a RETURN command instead of being silently swallowed.

**Architecture:** Test-driven, grounded in the real codebase (see `2026-05-31-try-catch-IMPLEMENTATION-NOTES.md` for the verified reality map). The engine gains a real per-step **fail signal** (`UserAction.action:'fail'` + `failure_mode`), CATCH/RETURN auto-completing handlers, an `active_catches` map, TRY routing, and RETURN command dispatch. The `web-ui` coordinator classifies an Action-Container terminal into `ERROR/ABORT/TIMEOUT`, adds a wall-clock timeout, and feeds the fail signal in (replacing the empty-submit stub at `WorkflowCoordinator.ts:494`). The canonical behaviour spec is `specs/2026-05-31-try-catch-return-design.md`.

**Tech Stack:** TypeScript, AJV 8, Node's built-in test runner (`node:test` + `node:assert/strict`), `tsx` (web-ui). NO Vitest.

**Scope boundaries:**
- TypeScript / web only. Kotlin `kmp-engine` parity is the sibling plan `2026-05-31-try-catch-runtime-native-kmp.md`.
- `release_on_catch` is **descoped** (user decision): accepted as an advisory flag; no auto-release (the resource manager has no holder tracking). Catch networks free resources via explicit Release commands.
- **Shared `spec/conformance` fixtures are NOT added here** — they are the capstone of the Kotlin plan, so neither engine's conformance suite is ever red at a plan boundary. This plan is verified entirely by `node:test` unit tests (the established pattern: `engine-restart.test.ts`, `validator.test.ts`, etc.).
- Action Container REST protocol unchanged.

---

## Test workflow (every task)

**Engine + validator tasks** — run from `C:/Trajectory/TrajectoryRuntime/engines/web`:
```
npm run build && node --test dist/<name>.test.js
```
(`tsc` compiles `src/*.test.ts` → `dist/`, then `node --test` runs the compiled file. Run the whole suite with `npm test`.)

**Coordinator tasks (Phase F)** — run from `C:/Trajectory/TrajectoryRuntime/engines/web-ui`:
```
node --import tsx --test src/<path>/<name>.test.ts
```
(`tsx` transpiles on the fly; no build step. Append `--test-name-pattern "<substr>"` to run one test.)

Commit after each task with the message shown in its final step.

---

## File Structure Plan

### Files to create
| File | Responsibility |
|---|---|
| `engines/web/src/catch-network-partition.ts` | Pure helper: BFS-from-CATCH partition `{mainFlowStepOids, catchNetworkStepOids, networksByCatchId}`. Shared by validator (orphan exemption + topology) and engine (routing/cleanup). |
| `engines/web/src/catch-network-partition.test.ts` | Unit tests for the partition helper. |
| `engines/web/src/try-catch-schema.test.ts` | Direct-AJV tests for the schema additions (Phase A). |
| `engines/web/src/try-catch-validator.test.ts` | Unit tests for every new validator rule (Phase B). |
| `engines/web/src/try-catch-handlers.test.ts` | Unit tests for `activateCatchStep` (Phase C). |
| `engines/web/src/try-catch-engine.test.ts` | Engine integration tests: fail signal, TRY routing, RETURN dispatch (Phases D–E). |
| `engines/web/src/return-dispatcher.ts` | The four RETURN command handlers (ABANDON/RESTART/GOTO/RETRY) + catch-network cleanup. |

### Files to modify
| File | Change |
|---|---|
| `spec/workflow-schema.json` | Add `CATCH`,`RETURN` to `StepType.enum`; add `$defs.TrySpecification`, `$defs.ReturnConfig`; add optional `try_specifications`/`catch_id`/`return_config` to `MasterWorkflowStep`. |
| `engines/web/src/types.ts` | Add `FailureMode`, `TrySpecification`, `ReturnConfig`, `CatchContext`; extend `UserAction` with `'fail'` + `failure_mode?` + `error?`; add `try_specifications?`/`catch_id?`/`return_config?` to `MasterWorkflowStep`. |
| `engines/web/src/validator.ts` | Exempt catch-network steps from `ORPHANED_STEP`; add new `tryCatchValidation` phase (structural/cross-ref/topology rules). |
| `engines/web/src/step-handlers.ts` | Add `CATCH`,`RETURN` to `isAutoCompleting`; add `activateCatchStep`. |
| `engines/web/src/engine.ts` | Add `active_catches` map; CATCH/RETURN branches in `activateStepAfterResources`; `'fail'` branch in `submitAction` (classify + TRY route or ERRORED); call `return-dispatcher`. |
| `engines/web-ui/src/actionProxy/stateMapping.ts` | Add `mapServerStateToFailureMode`. |
| `engines/web-ui/src/actionProxy/ActionProxyController.ts` | Widen `onTerminal` payload with `failureMode`; classify in `emitTerminal`; add injectable wall-clock timeout → `sendCommand('ABORT')`. |
| `engines/web-ui/src/coordinator/WorkflowCoordinator.ts` | Widen `onTerminal`; replace the empty-submit stub (`:494`) with `submitAction({action:'fail', failure_mode, error})`; pass `timeoutMs`. |

### Files NOT touched
`engines/web/src/loader.ts` (new fields ride through). `engines/web/src/runner.ts` + `spec/conformance/*` (conformance fixtures are the Kotlin plan's capstone). The Action Container.

---

## Phase A — Schema additions

The schema lands first. Tested via a direct-AJV harness (the new step types can't be reached through `validate()` until the validator's orphan exemption exists in Phase B, so we test the schema layer in isolation here).

### Task A1: Add CATCH and RETURN to the step_type enum

**Files:**
- Create: `engines/web/src/try-catch-schema.test.ts`
- Modify: `spec/workflow-schema.json` (`$defs.StepType.enum`, lines 21-25)

- [ ] **Step 1: Write the failing test**

Create `engines/web/src/try-catch-schema.test.ts`:
```typescript
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';
import Ajv from 'ajv';
import { readFileSync } from 'node:fs';
import { resolve, dirname } from 'node:path';
import { fileURLToPath } from 'node:url';

const __dirname = dirname(fileURLToPath(import.meta.url));

// Mirror validator.ts loadSchema()'s candidate resolution (from dist/).
function loadSchema(): Record<string, unknown> {
  const candidates = [
    resolve(__dirname, '../../../../spec/workflow-schema.json'),
    resolve(__dirname, '../../../spec/workflow-schema.json'),
    resolve(__dirname, '../../spec/workflow-schema.json'),
  ];
  for (const p of candidates) {
    try {
      const raw = JSON.parse(readFileSync(p, 'utf-8')) as Record<string, unknown>;
      delete raw['$schema'];
      return raw;
    } catch { /* next */ }
  }
  throw new Error('schema not found');
}

function compile() {
  const ajv = new (Ajv as unknown as typeof Ajv.default)({ allErrors: true, strict: false });
  return ajv.compile(loadSchema());
}

const DATE = '2026-05-31T12:00:00.000Z';
function wfWithStep(step: Record<string, unknown>) {
  return {
    local_id: 'wf', oid: 'wf-1', version: '1.0.0', last_modified_date: DATE,
    steps: [
      { local_id: 'Start', oid: 's1', step_type: 'START', version: '1.0.0', last_modified_date: DATE },
      { local_id: 'End', oid: 's2', step_type: 'END', version: '1.0.0', last_modified_date: DATE },
      step,
    ],
    connections: [{ from_step_id: 's1', to_step_id: 's2' }],
  };
}

describe('schema: CATCH/RETURN step types', () => {
  it('accepts a CATCH step type', () => {
    const validate = compile();
    const ok = validate(wfWithStep({
      local_id: 'C', oid: 'c1', step_type: 'CATCH', version: '1.0.0', last_modified_date: DATE, catch_id: 'X',
    }));
    const enumErr = (validate.errors ?? []).some(e => e.keyword === 'enum' && (e.instancePath ?? '').includes('step_type'));
    assert.equal(enumErr, false, `unexpected enum error: ${JSON.stringify(validate.errors)}`);
    assert.equal(ok, true);
  });

  it('accepts a RETURN step type', () => {
    const validate = compile();
    const ok = validate(wfWithStep({
      local_id: 'R', oid: 'r1', step_type: 'RETURN', version: '1.0.0', last_modified_date: DATE,
      return_config: { command: 'ABANDON' },
    }));
    assert.equal(ok, true, `errors: ${JSON.stringify(validate.errors)}`);
  });

  it('still rejects a genuinely unknown step type (control)', () => {
    const validate = compile();
    validate(wfWithStep({ local_id: 'B', oid: 'b1', step_type: 'BOGUS', version: '1.0.0', last_modified_date: DATE }));
    const enumErr = (validate.errors ?? []).some(e => e.keyword === 'enum' && (e.instancePath ?? '').includes('step_type'));
    assert.equal(enumErr, true);
  });
});
```

- [ ] **Step 2: Run — expect failure**

`npm run build && node --test dist/try-catch-schema.test.js`
Expected: the CATCH/RETURN tests FAIL with an `enum` error on `step_type` (the control test passes).

- [ ] **Step 3: Add the enum values**

In `spec/workflow-schema.json`, `$defs.StepType.enum` (lines 21-25), append `"CATCH"`, `"RETURN"`:
```json
    "StepType": {
      "type": "string",
      "enum": [
        "START", "END", "ACTION PROXY", "WORKFLOW PROXY", "SELECT 1",
        "WAIT ANY", "PARALLEL", "WAIT ALL", "MATH", "SCRIPT",
        "YES_NO", "USER_INTERACTION", "CATCH", "RETURN"
      ]
    },
```

- [ ] **Step 4: Run — expect pass**

`npm run build && node --test dist/try-catch-schema.test.js` → all 3 pass.

- [ ] **Step 5: Commit**

```
git add spec/workflow-schema.json engines/web/src/try-catch-schema.test.ts
git commit -m "feat(schema): add CATCH and RETURN to step_type enum"
```

### Task A2: Add TrySpecification and ReturnConfig $defs and wire optional fields

**Files:**
- Modify: `spec/workflow-schema.json` (`$defs`; `MasterWorkflowStep.properties`)
- Modify: `engines/web/src/try-catch-schema.test.ts`

- [ ] **Step 1: Add failing tests**

Append to `try-catch-schema.test.ts`:
```typescript
describe('schema: try_specifications / return_config shape', () => {
  it('accepts a well-formed try_specifications array on an action step', () => {
    const validate = compile();
    const ok = validate(wfWithStep({
      local_id: 'A', oid: 'a1', step_type: 'ACTION PROXY', version: '1.0.0', last_modified_date: DATE,
      try_specifications: [
        { mode: 'ERROR', catch_id: 'C1', release_on_catch: true },
        { mode: 'TIMEOUT', catch_id: 'C1' },
      ],
    }));
    assert.equal(ok, true, `errors: ${JSON.stringify(validate.errors)}`);
  });

  it('rejects a try_specifications entry with an unknown mode', () => {
    const validate = compile();
    validate(wfWithStep({
      local_id: 'A', oid: 'a1', step_type: 'ACTION PROXY', version: '1.0.0', last_modified_date: DATE,
      try_specifications: [{ mode: 'BOGUS', catch_id: 'C1' }],
    }));
    const modeErr = (validate.errors ?? []).some(e => e.keyword === 'enum' && (e.instancePath ?? '').includes('mode'));
    assert.equal(modeErr, true, `errors: ${JSON.stringify(validate.errors)}`);
  });

  it('rejects a return_config with an unknown command', () => {
    const validate = compile();
    validate(wfWithStep({
      local_id: 'R', oid: 'r1', step_type: 'RETURN', version: '1.0.0', last_modified_date: DATE,
      return_config: { command: 'NOPE' },
    }));
    const cmdErr = (validate.errors ?? []).some(e => e.keyword === 'enum' && (e.instancePath ?? '').includes('command'));
    assert.equal(cmdErr, true, `errors: ${JSON.stringify(validate.errors)}`);
  });
});
```

- [ ] **Step 2: Run — expect failure** (`npm run build && node --test dist/try-catch-schema.test.js`). The malformed-mode/command tests fail because the `$defs` don't exist yet (extra props are tolerated, so no error is raised).

- [ ] **Step 3: Add the `$defs`** — in `spec/workflow-schema.json`, add after `ScriptConfig` (line 390):
```json
    "TrySpecification": {
      "type": "object",
      "required": ["mode", "catch_id"],
      "properties": {
        "mode": { "type": "string", "enum": ["ERROR", "ABORT", "TIMEOUT"] },
        "catch_id": { "type": "string", "minLength": 1 },
        "release_on_catch": { "type": "boolean" }
      }
    },
    "ReturnConfig": {
      "type": "object",
      "required": ["command"],
      "properties": {
        "command": { "type": "string", "enum": ["ABANDON", "RESTART", "GOTO", "RETRY"] },
        "restart_mode": { "type": "string", "enum": ["CLEAN", "KEEP"] },
        "goto_step_oid": { "type": "string" }
      }
    },
```

- [ ] **Step 4: Wire onto `MasterWorkflowStep`** — in the `properties` block (after `connection_references`, line 582), add:
```json
            "try_specifications": {
              "type": "array",
              "items": { "$ref": "#/$defs/TrySpecification" }
            },
            "catch_id": { "type": "string", "minLength": 1 },
            "return_config": { "$ref": "#/$defs/ReturnConfig" }
```
(Insert a comma after the `connection_references` array def so JSON stays valid. These are optional — do NOT add to `required`. Semantic rules — "try only on action steps", "return_config only on RETURN" — are enforced in the validator, Phase B, because the schema has no per-step-type conditionals.)

- [ ] **Step 5: Run — expect pass** (`npm run build && node --test dist/try-catch-schema.test.js`, all pass).

- [ ] **Step 6: Commit**
```
git add spec/workflow-schema.json engines/web/src/try-catch-schema.test.ts
git commit -m "feat(schema): add TrySpecification and ReturnConfig defs"
```

---

## Phase B — Validator rules

The semantic validator (`validate()`, fail-fast single-error) must (1) stop flagging catch-network islands as `ORPHANED_STEP`, and (2) enforce the spec §6 rules. A new `tryCatchValidation` phase runs **after** `semanticValidation` and **before** the AJV phase, so its specific codes (e.g. `INVALID_TRY_MODE`) win over generic AJV errors.

### Task B1: Catch-network partition helper

**Files:**
- Create: `engines/web/src/catch-network-partition.ts`
- Create: `engines/web/src/catch-network-partition.test.ts`

- [ ] **Step 1: Write failing tests**

Create `engines/web/src/catch-network-partition.test.ts`:
```typescript
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';
import { partitionCatchNetworks } from './catch-network-partition.js';

const steps = [
  { oid: 's1', step_type: 'START' },
  { oid: 's2', step_type: 'ACTION PROXY' },
  { oid: 's3', step_type: 'END' },
  { oid: 'c1', step_type: 'CATCH', catch_id: 'C1' },
  { oid: 'c2', step_type: 'USER_INTERACTION' },
  { oid: 'c3', step_type: 'RETURN' },
];
const connections = [
  { from_step_id: 's1', to_step_id: 's2' },
  { from_step_id: 's2', to_step_id: 's3' },
  { from_step_id: 'c1', to_step_id: 'c2' },
  { from_step_id: 'c2', to_step_id: 'c3' },
];

describe('partitionCatchNetworks', () => {
  it('separates main-flow from catch-network steps', () => {
    const p = partitionCatchNetworks(steps, connections);
    assert.deepEqual([...p.mainFlowStepOids].sort(), ['s1', 's2', 's3']);
    assert.deepEqual([...p.catchNetworkStepOids].sort(), ['c1', 'c2', 'c3']);
  });

  it('indexes networks by catch_id', () => {
    const p = partitionCatchNetworks(steps, connections);
    assert.deepEqual([...(p.networksByCatchId.get('C1') ?? [])].sort(), ['c1', 'c2', 'c3']);
  });

  it('excludes a CATCH whose downstream has no RETURN', () => {
    const p = partitionCatchNetworks(
      [{ oid: 's1', step_type: 'START' }, { oid: 's2', step_type: 'END' },
       { oid: 'c1', step_type: 'CATCH', catch_id: 'C1' }, { oid: 'c2', step_type: 'USER_INTERACTION' }],
      [{ from_step_id: 's1', to_step_id: 's2' }, { from_step_id: 'c1', to_step_id: 'c2' }],
    );
    assert.equal(p.catchNetworkStepOids.size, 0);
    assert.equal(p.mainFlowStepOids.size, 4);
  });
});
```

- [ ] **Step 2: Run — expect failure** (`npm run build` fails: module not found).

- [ ] **Step 3: Implement** — create `engines/web/src/catch-network-partition.ts`:
```typescript
// Copyright (c) 2026 Saturnis.io. All rights reserved.
// Licensed under the GNU AGPL v3. See LICENSE.md for details.

export interface PartitionStep { oid: string; step_type: string; catch_id?: string }
export interface PartitionConnection { from_step_id: string; to_step_id: string }

export interface CatchNetworkPartition {
  mainFlowStepOids: Set<string>;
  catchNetworkStepOids: Set<string>;
  /** catch_id → set of step oids reachable from that CATCH (only if the subgraph contains a RETURN). */
  networksByCatchId: Map<string, Set<string>>;
}

/** Spec §6.5: a catch network is the BFS-closure from a CATCH over outgoing edges, but only counts if it reaches a RETURN. */
export function partitionCatchNetworks(
  steps: PartitionStep[],
  connections: PartitionConnection[],
): CatchNetworkPartition {
  const outEdges = new Map<string, string[]>();
  for (const c of connections) {
    const list = outEdges.get(c.from_step_id);
    if (list) list.push(c.to_step_id);
    else outEdges.set(c.from_step_id, [c.to_step_id]);
  }
  const typeByOid = new Map<string, string>();
  for (const s of steps) typeByOid.set(s.oid, s.step_type);

  const networksByCatchId = new Map<string, Set<string>>();
  const catchNetworkStepOids = new Set<string>();

  for (const step of steps) {
    if (step.step_type !== 'CATCH' || !step.catch_id) continue;
    const visited = new Set<string>();
    const queue: string[] = [step.oid];
    let hasReturn = false;
    while (queue.length > 0) {
      const cur = queue.shift()!;
      if (visited.has(cur)) continue;
      visited.add(cur);
      if (typeByOid.get(cur) === 'RETURN') hasReturn = true;
      for (const n of outEdges.get(cur) ?? []) {
        if (!visited.has(n)) queue.push(n);
      }
    }
    if (hasReturn) {
      networksByCatchId.set(step.catch_id, visited);
      for (const oid of visited) catchNetworkStepOids.add(oid);
    }
  }

  const mainFlowStepOids = new Set<string>();
  for (const s of steps) {
    if (!catchNetworkStepOids.has(s.oid)) mainFlowStepOids.add(s.oid);
  }
  return { mainFlowStepOids, catchNetworkStepOids, networksByCatchId };
}
```

- [ ] **Step 4: Run — expect pass.**

- [ ] **Step 5: Commit**
```
git add engines/web/src/catch-network-partition.ts engines/web/src/catch-network-partition.test.ts
git commit -m "feat(runtime): catch-network partition helper"
```

### Task B2: Exempt catch networks from the orphan check (minimal valid CATCH/RETURN workflow)

**Files:**
- Modify: `engines/web/src/validator.ts` (`semanticValidation`, orphan loop lines 126-130)
- Create: `engines/web/src/try-catch-validator.test.ts`

- [ ] **Step 1: Write the failing test**

Create `engines/web/src/try-catch-validator.test.ts`:
```typescript
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';
import { validate } from './validator.js';

const DATE = '2026-05-31T12:00:00.000Z';
function step(o: Record<string, unknown>) {
  return { version: '1.0.0', last_modified_date: DATE, ...o };
}
/** START→Action→END main flow + an independent CATCH→RETURN island. */
function baseWithCatch(extra: { trySpec?: unknown; returnConfig?: unknown } = {}) {
  return {
    local_id: 'wf', oid: 'wf-1', version: '1.0.0', last_modified_date: DATE, schemaVersion: '4.0',
    steps: [
      step({ local_id: 'Start', oid: 's1', step_type: 'START' }),
      step({ local_id: 'Action', oid: 's2', step_type: 'ACTION PROXY',
             try_specifications: extra.trySpec ?? [{ mode: 'ERROR', catch_id: 'C1' }] }),
      step({ local_id: 'End', oid: 's3', step_type: 'END' }),
      step({ local_id: 'Catch', oid: 'c1', step_type: 'CATCH', catch_id: 'C1' }),
      step({ local_id: 'Ret', oid: 'r1', step_type: 'RETURN',
             return_config: extra.returnConfig ?? { command: 'ABANDON' } }),
    ],
    connections: [
      { from_step_id: 's1', to_step_id: 's2' },
      { from_step_id: 's2', to_step_id: 's3' },
      { from_step_id: 'c1', to_step_id: 'r1' },
    ],
  };
}

describe('validator: catch-network is not orphaned', () => {
  it('accepts a minimal well-formed TRY/CATCH/RETURN workflow', () => {
    const r = validate(baseWithCatch());
    assert.equal(r.valid, true, `expected valid, got ${r.error_code}: ${r.error_message}`);
  });
});
```

- [ ] **Step 2: Run — expect failure** (`node --test dist/try-catch-validator.test.js`): returns `valid:false` with `ORPHANED_STEP` (the catch island is unreachable from START).

- [ ] **Step 3: Implement the exemption** — in `validator.ts`:

At the top of the file, add the import (next to the existing imports, ~line 7):
```typescript
import { partitionCatchNetworks, type PartitionStep, type PartitionConnection } from './catch-network-partition.js';
```
In `semanticValidation`, replace the orphan loop (lines 126-130):
```typescript
  for (const step of steps) {
    if (!reachable.has(step.oid as string)) {
      return { valid: false, error_code: 'ORPHANED_STEP', error_message: `Step ${step.oid} is not reachable from START` };
    }
  }
```
with:
```typescript
  // Catch networks are intentional disconnected islands (reached at runtime via TRY, not via a connection).
  const partition = partitionCatchNetworks(
    steps as unknown as PartitionStep[],
    connections as unknown as PartitionConnection[],
  );
  for (const step of steps) {
    const oid = step.oid as string;
    if (partition.catchNetworkStepOids.has(oid)) continue;
    if (!reachable.has(oid)) {
      return { valid: false, error_code: 'ORPHANED_STEP', error_message: `Step ${oid} is not reachable from START` };
    }
  }
```

- [ ] **Step 4: Run — expect pass.**

- [ ] **Step 5: Commit**
```
git add engines/web/src/validator.ts engines/web/src/try-catch-validator.test.ts
git commit -m "feat(validator): exempt catch networks from orphan check"
```

### Task B3: Structural TRY/CATCH/RETURN rules

**Files:**
- Modify: `engines/web/src/validator.ts` (new `tryCatchValidation` phase + wire into `validate()`)
- Modify: `engines/web/src/try-catch-validator.test.ts`

- [ ] **Step 1: Add failing tests** — append to `try-catch-validator.test.ts`:
```typescript
describe('validator: structural TRY rules', () => {
  function withExtraStep(s: Record<string, unknown>, conns: Array<Record<string, string>> = []) {
    const wf = baseWithCatch();
    wf.steps.push(step(s) as never);
    wf.connections.push(...conns);
    return wf;
  }

  it('CATCH_WRONG_DEGREE when a CATCH has an incoming connection', () => {
    const wf = baseWithCatch();
    wf.connections.push({ from_step_id: 's1', to_step_id: 'c1' });
    assert.equal(validate(wf).error_code, 'CATCH_WRONG_DEGREE');
  });
  it('RETURN_WRONG_DEGREE when a RETURN has an outgoing connection', () => {
    const wf = baseWithCatch();
    wf.connections.push({ from_step_id: 'r1', to_step_id: 's3' });
    assert.equal(validate(wf).error_code, 'RETURN_WRONG_DEGREE');
  });
  it('TRY_ON_INVALID_STEP when try_specifications is on a USER_INTERACTION', () => {
    const wf = withExtraStep(
      { local_id: 'U', oid: 'u1', step_type: 'USER_INTERACTION', try_specifications: [{ mode: 'ERROR', catch_id: 'C1' }] },
      [{ from_step_id: 's2', to_step_id: 'u1' }, { from_step_id: 'u1', to_step_id: 's3' }],
    );
    // remove the now-redundant s2→s3 edge so u1 is on the main path
    wf.connections = wf.connections.filter(c => !(c.from_step_id === 's2' && c.to_step_id === 's3'));
    assert.equal(validate(wf).error_code, 'TRY_ON_INVALID_STEP');
  });
  it('DUPLICATE_TRY_MODE when one step repeats a mode', () => {
    const r = validate(baseWithCatch({ trySpec: [{ mode: 'ERROR', catch_id: 'C1' }, { mode: 'ERROR', catch_id: 'C1' }] }));
    assert.equal(r.error_code, 'DUPLICATE_TRY_MODE');
  });
  it('MISSING_RESTART_MODE when RESTART has no restart_mode', () => {
    assert.equal(validate(baseWithCatch({ returnConfig: { command: 'RESTART' } })).error_code, 'MISSING_RESTART_MODE');
  });
  it('MISSING_GOTO_TARGET when GOTO has no goto_step_oid', () => {
    assert.equal(validate(baseWithCatch({ returnConfig: { command: 'GOTO' } })).error_code, 'MISSING_GOTO_TARGET');
  });
});
```

- [ ] **Step 2: Run — expect failures.**

- [ ] **Step 3: Implement `tryCatchValidation`** — in `validator.ts`, add the function (near `semanticValidation`):
```typescript
function tryCatchValidation(workflow: Record<string, unknown>): ValidationResult | null {
  const steps = (workflow['steps'] as Record<string, unknown>[]) ?? [];
  const connections = (workflow['connections'] as Record<string, unknown>[]) ?? [];
  const fail = (code: string, msg: string, oid?: string): ValidationResult =>
    ({ valid: false, error_code: code, error_message: oid ? `${msg} (step ${oid})` : msg });

  const TRY_SCOPED = new Set(['ACTION PROXY', 'WAIT ACTION PROXY']);
  const MODES = new Set(['ERROR', 'ABORT', 'TIMEOUT']);
  const COMMANDS = new Set(['ABANDON', 'RESTART', 'GOTO', 'RETRY']);

  for (const step of steps) {
    const type = normalizeStepType(String(step.step_type));
    const oid = step.oid as string;
    const inDeg = connections.filter(c => c.to_step_id === oid).length;
    const outDeg = connections.filter(c => c.from_step_id === oid).length;

    if (type === 'CATCH' && (inDeg !== 0 || outDeg !== 1)) {
      return fail('CATCH_WRONG_DEGREE', `CATCH must have 0 incoming and 1 outgoing connection (has ${inDeg} in, ${outDeg} out)`, oid);
    }
    if (type === 'RETURN' && (inDeg !== 1 || outDeg !== 0)) {
      return fail('RETURN_WRONG_DEGREE', `RETURN must have 1 incoming and 0 outgoing connections (has ${inDeg} in, ${outDeg} out)`, oid);
    }

    const tries = step.try_specifications as Array<{ mode: string; catch_id: string }> | undefined;
    if (tries && tries.length > 0) {
      if (!TRY_SCOPED.has(type)) {
        return fail('TRY_ON_INVALID_STEP', `try_specifications not allowed on step_type '${type}'`, oid);
      }
      const seen = new Set<string>();
      for (const t of tries) {
        if (!MODES.has(t.mode)) return fail('INVALID_TRY_MODE', `invalid try mode '${t.mode}'`, oid);
        if (seen.has(t.mode)) return fail('DUPLICATE_TRY_MODE', `mode '${t.mode}' appears more than once`, oid);
        seen.add(t.mode);
      }
    }

    if (type === 'RETURN') {
      const rc = step.return_config as { command?: string; restart_mode?: string; goto_step_oid?: string } | undefined;
      if (rc) {
        if (!rc.command || !COMMANDS.has(rc.command)) return fail('INVALID_RETURN_COMMAND', `invalid RETURN command '${rc.command}'`, oid);
        if (rc.command === 'RESTART' && !rc.restart_mode) return fail('MISSING_RESTART_MODE', `RESTART requires restart_mode`, oid);
        if (rc.restart_mode && rc.restart_mode !== 'CLEAN' && rc.restart_mode !== 'KEEP') return fail('INVALID_RESTART_MODE', `invalid restart_mode '${rc.restart_mode}'`, oid);
        if (rc.command === 'GOTO' && !rc.goto_step_oid) return fail('MISSING_GOTO_TARGET', `GOTO requires goto_step_oid`, oid);
      }
    }
  }
  return null;
}
```
Wire it into `validate()` immediately after the `semanticValidation` block (after line 562):
```typescript
  // Phase A1.5: TRY/CATCH/RETURN structural + cross-reference + topology rules
  const tryCatchError = tryCatchValidation(workflow);
  if (tryCatchError) return tryCatchError;
```

- [ ] **Step 4: Run — expect pass.**

- [ ] **Step 5: Commit**
```
git add engines/web/src/validator.ts engines/web/src/try-catch-validator.test.ts
git commit -m "feat(validator): structural TRY/CATCH/RETURN rules"
```

### Task B4: Cross-reference rules

**Files:** Modify `validator.ts` (extend `tryCatchValidation`) and `try-catch-validator.test.ts`.

- [ ] **Step 1: Add failing tests** — append:
```typescript
describe('validator: cross-reference TRY rules', () => {
  it('DUPLICATE_CATCH_ID', () => {
    const wf = baseWithCatch();
    wf.steps.push(step({ local_id: 'C2', oid: 'c2', step_type: 'CATCH', catch_id: 'C1' }) as never,
                  step({ local_id: 'R2', oid: 'r2', step_type: 'RETURN', return_config: { command: 'ABANDON' } }) as never);
    wf.connections.push({ from_step_id: 'c2', to_step_id: 'r2' });
    assert.equal(validate(wf).error_code, 'DUPLICATE_CATCH_ID');
  });
  it('UNMATCHED_TRY', () => {
    assert.equal(validate(baseWithCatch({ trySpec: [{ mode: 'ERROR', catch_id: 'Ghost' }] })).error_code, 'UNMATCHED_TRY');
  });
  it('GOTO_TARGET_NOT_FOUND', () => {
    assert.equal(validate(baseWithCatch({ returnConfig: { command: 'GOTO', goto_step_oid: 'ghost' } })).error_code, 'GOTO_TARGET_NOT_FOUND');
  });
  it('GOTO_TARGET_IN_CATCH', () => {
    // GOTO points at the CATCH's own step c1 (inside a catch network)
    assert.equal(validate(baseWithCatch({ returnConfig: { command: 'GOTO', goto_step_oid: 'c1' } })).error_code, 'GOTO_TARGET_IN_CATCH');
  });
});
```

- [ ] **Step 2: Run — expect failures.**

- [ ] **Step 3: Implement** — in `tryCatchValidation`, before the final `return null;`, add:
```typescript
  // Cross-reference: catch_id uniqueness, TRY→CATCH resolution, GOTO target.
  const catchIds = new Set<string>();
  for (const step of steps) {
    if (normalizeStepType(String(step.step_type)) !== 'CATCH') continue;
    const cid = step.catch_id as string | undefined;
    if (!cid) continue;
    if (catchIds.has(cid)) return fail('DUPLICATE_CATCH_ID', `catch_id '${cid}' used by more than one CATCH`, step.oid as string);
    catchIds.add(cid);
  }
  const oidSet = new Set(steps.map(s => s.oid as string));
  const partition = partitionCatchNetworks(steps as unknown as PartitionStep[], connections as unknown as PartitionConnection[]);
  for (const step of steps) {
    const tries = step.try_specifications as Array<{ catch_id: string }> | undefined;
    for (const t of tries ?? []) {
      if (!catchIds.has(t.catch_id)) return fail('UNMATCHED_TRY', `TRY references undefined catch_id '${t.catch_id}'`, step.oid as string);
    }
    if (normalizeStepType(String(step.step_type)) === 'RETURN') {
      const rc = step.return_config as { command?: string; goto_step_oid?: string } | undefined;
      if (rc?.command === 'GOTO' && rc.goto_step_oid) {
        if (!oidSet.has(rc.goto_step_oid)) return fail('GOTO_TARGET_NOT_FOUND', `GOTO target '${rc.goto_step_oid}' not found`, step.oid as string);
        if (partition.catchNetworkStepOids.has(rc.goto_step_oid)) return fail('GOTO_TARGET_IN_CATCH', `GOTO target '${rc.goto_step_oid}' is inside a catch network`, step.oid as string);
      }
    }
  }
```

- [ ] **Step 4: Run — expect pass.**

- [ ] **Step 5: Commit**
```
git add engines/web/src/validator.ts engines/web/src/try-catch-validator.test.ts
git commit -m "feat(validator): cross-reference TRY/CATCH/RETURN rules"
```

### Task B5: Topology rules + ORPHANED_CATCH accepted

**Files:** Modify `validator.ts` and `try-catch-validator.test.ts`.

- [ ] **Step 1: Add failing tests** — append:
```typescript
describe('validator: topology TRY rules', () => {
  it('CROSS_NETWORK_EDGE when an edge crosses the boundary', () => {
    const wf = baseWithCatch();
    wf.connections.push({ from_step_id: 's2', to_step_id: 'r1' }); // main → catch-network
    assert.equal(validate(wf).error_code, 'CROSS_NETWORK_EDGE');
  });
  it('CATCH_WITHOUT_RETURN when a CATCH cannot reach a RETURN', () => {
    const wf = baseWithCatch();
    // replace the CATCH's RETURN target with a non-RETURN dead end
    wf.steps = wf.steps.map(s => (s as Record<string,unknown>).oid === 'r1'
      ? step({ local_id: 'U', oid: 'r1', step_type: 'USER_INTERACTION' }) as never : s);
    assert.equal(validate(wf).error_code, 'CATCH_WITHOUT_RETURN');
  });
  it('accepts an ORPHANED_CATCH (catch_id referenced by no TRY) — valid:true', () => {
    const wf = baseWithCatch({ trySpec: [] }); // action has no TRY → C1 is orphaned
    const r = validate(wf);
    assert.equal(r.valid, true, `expected valid, got ${r.error_code}`);
  });
});
```

- [ ] **Step 2: Run — expect failures** (the first two return the wrong/no code; the third may currently fail for another reason — confirm it ends `valid:true` after impl).

- [ ] **Step 3: Implement** — in `tryCatchValidation`, before `return null;`, add (reuse the `partition` computed in B4):
```typescript
  // Topology: no edge may cross the main-flow / catch-network boundary.
  for (const c of connections) {
    const fromIn = partition.catchNetworkStepOids.has(c.from_step_id as string);
    const toIn = partition.catchNetworkStepOids.has(c.to_step_id as string);
    if (fromIn !== toIn) {
      return fail('CROSS_NETWORK_EDGE', `connection ${c.from_step_id} → ${c.to_step_id} crosses the catch-network boundary`);
    }
  }
  // Every CATCH must reach a RETURN (else it's not in networksByCatchId).
  for (const step of steps) {
    if (normalizeStepType(String(step.step_type)) !== 'CATCH') continue;
    const cid = step.catch_id as string | undefined;
    if (cid && !partition.networksByCatchId.has(cid)) {
      return fail('CATCH_WITHOUT_RETURN', `CATCH '${step.local_id}' has no reachable RETURN`, step.oid as string);
    }
  }
  // ORPHANED_CATCH (catch_id referenced by no TRY) is accepted at runtime (valid:true) — editor surfaces the warning.
```
(Note: `RETURN_WITHOUT_CATCH` and `CATCH_NETWORK_NOT_CONNECTED` from spec §6.3 are covered transitively — a RETURN not in any catch network falls outside `catchNetworkStepOids`, so any edge into it from a CATCH-reachable node would have been caught by the degree/cross-network checks; a disconnected RETURN with the required 1-in-degree must have a CATCH-reachable predecessor. Add explicit codes only if a fixture surfaces a gap.)

- [ ] **Step 4: Run — expect pass.**

- [ ] **Step 5: Commit**
```
git add engines/web/src/validator.ts engines/web/src/try-catch-validator.test.ts
git commit -m "feat(validator): topology TRY/CATCH/RETURN rules; accept orphaned CATCH"
```

---

## Phase C — Types + CATCH/RETURN auto-complete handlers

### Task C1: Types

**Files:** Modify `engines/web/src/types.ts`.

- [ ] **Step 1: Add the declarations** — append to `types.ts` (after `MasterWorkflowStep`/near the runtime types):
```typescript
export type FailureMode = 'ERROR' | 'ABORT' | 'TIMEOUT';

export interface TrySpecification {
  mode: FailureMode;
  catch_id: string;
  release_on_catch?: boolean;
}
export interface ReturnConfig {
  command: 'ABANDON' | 'RESTART' | 'GOTO' | 'RETRY';
  restart_mode?: 'CLEAN' | 'KEEP';
  goto_step_oid?: string;
}
export interface CatchContext {
  catch_oid: string;
  trigger_step_oid: string;
  trigger_step_name: string;
  trigger_reason: FailureMode;
  error_message: string | null;
  activated_at: string;
}
```
Extend `MasterWorkflowStep` (lines 348-360) with the optional fields:
```typescript
  try_specifications?: TrySpecification[];
  catch_id?: string;
  return_config?: ReturnConfig;
```
Extend `UserAction` (lines 432-437):
```typescript
export interface UserAction {
  step_oid: string;
  action: 'submit' | 'button_press' | 'yes' | 'no' | 'pause' | 'resume' | 'fail';
  form_values?: Record<string, unknown>;
  button_output?: string;
  /** Set when action === 'fail': the classified failure mode driving TRY dispatch. */
  failure_mode?: FailureMode;
  /** Set when action === 'fail': the underlying error message, if any. */
  error?: string;
}
```

- [ ] **Step 2: Verify it compiles** — `npm run build` (no test needed; this is a type-only change consumed by later tasks). Expected: clean build.

- [ ] **Step 3: Commit**
```
git add engines/web/src/types.ts
git commit -m "feat(types): TRY/CATCH/RETURN types + fail UserAction variant"
```

### Task C2: CATCH/RETURN auto-completing + activateCatchStep

**Files:**
- Modify: `engines/web/src/step-handlers.ts`
- Create: `engines/web/src/try-catch-handlers.test.ts`

- [ ] **Step 1: Write the failing test** — create `engines/web/src/try-catch-handlers.test.ts`:
```typescript
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';
import { isAutoCompleting, activateCatchStep } from './step-handlers.js';
import { PropertyStore } from './properties.js';
import type { CatchContext, MasterWorkflowStep } from './types.js';

describe('CATCH/RETURN handlers', () => {
  it('recognises CATCH and RETURN as auto-completing', () => {
    assert.equal(isAutoCompleting('CATCH'), true);
    assert.equal(isAutoCompleting('RETURN'), true);
  });

  it('writes trigger info to declared output_parameter_specifications targets', () => {
    const store = new PropertyStore();
    store.set('FailureContext.Mode', '');
    store.set('FailureContext.Message', '');
    const step = {
      oid: 'c1', step_type: 'CATCH', local_id: 'Catch', catch_id: 'C1',
      output_parameter_specifications: [
        { id: 'trigger_reason', target: 'FailureContext.Mode' },
        { id: 'error_message', target: 'FailureContext.Message' },
      ],
    } as unknown as MasterWorkflowStep;
    const ctx: CatchContext = {
      catch_oid: 'c1', trigger_step_oid: 'a1', trigger_step_name: 'Heat Oven',
      trigger_reason: 'ERROR', error_message: 'thermocouple failure', activated_at: '2026-05-31T12:00:00Z',
    };
    activateCatchStep(step, ctx, store);
    assert.equal(store.get('FailureContext.Mode'), 'ERROR');
    assert.equal(store.get('FailureContext.Message'), 'thermocouple failure');
  });

  it('ignores unknown output ids without throwing', () => {
    const store = new PropertyStore();
    const step = { oid: 'c1', step_type: 'CATCH', local_id: 'C', catch_id: 'C1',
      output_parameter_specifications: [{ id: 'not_a_field', target: 'X.Y' }] } as unknown as MasterWorkflowStep;
    const ctx: CatchContext = { catch_oid: 'c1', trigger_step_oid: 'a1', trigger_step_name: 'X',
      trigger_reason: 'ABORT', error_message: null, activated_at: '2026-05-31T12:00:00Z' };
    assert.doesNotThrow(() => activateCatchStep(step, ctx, store));
  });
});
```

- [ ] **Step 2: Run — expect failure** (build fails: `activateCatchStep` not exported).

- [ ] **Step 3: Implement** — in `step-handlers.ts`:

Update `isAutoCompleting` (lines 21-24):
```typescript
export function isAutoCompleting(stepType: string): boolean {
  const t = canonicalStepType(stepType);
  return ['START', 'END', 'PARALLEL', 'WAIT ANY', 'SELECT 1', 'SCRIPT', 'MATH', 'CATCH', 'RETURN'].includes(t);
}
```
Add the import for `CatchContext` to the type import block (lines 3-11) and append the handler:
```typescript
const KNOWN_CATCH_FIELDS: Record<string, (c: CatchContext) => string> = {
  trigger_step: c => c.trigger_step_name,
  trigger_step_oid: c => c.trigger_step_oid,
  trigger_reason: c => c.trigger_reason,
  error_message: c => c.error_message ?? '',
};

/** Spec §1.2 / §3.1: write the runtime-supplied trigger info to the CATCH's declared Value Property targets. */
export function activateCatchStep(
  step: MasterWorkflowStep,
  ctx: CatchContext,
  propertyStore: PropertyStore,
): void {
  for (const out of step.output_parameter_specifications ?? []) {
    const field = KNOWN_CATCH_FIELDS[out.id];
    if (!field || !out.target) continue; // unknown id: ignore silently (spec §1.2)
    propertyStore.set(out.target, field(ctx));
  }
}
```

- [ ] **Step 4: Run — expect pass.**

- [ ] **Step 5: Commit**
```
git add engines/web/src/step-handlers.ts engines/web/src/try-catch-handlers.test.ts
git commit -m "feat(engine): CATCH/RETURN auto-completing + activateCatchStep handler"
```

---

## Phase D — Engine: fail signal + TRY routing

All engine tests live in `engines/web/src/try-catch-engine.test.ts`. Shared fixture (define once at the top of the file):
```typescript
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';
import { WorkflowEngine } from './engine.js';
import type { MasterWorkflowSpecification } from './types.js';

const DATE = '2026-05-31T12:00:00.000Z';
function s(o: Record<string, unknown>) { return { version: '1.0.0', last_modified_date: DATE, ...o }; }

/** START→Action→END, with the Action's TRY (default ERROR→C1) and a CATCH→...→RETURN island. */
function tryWorkflow(opts: {
  trySpec?: unknown; catchOutputs?: unknown; returnConfig?: unknown; extraCatchSteps?: unknown[]; extraCatchConns?: unknown[];
  valueProps?: unknown;
} = {}): MasterWorkflowSpecification {
  return {
    local_id: 'wf', oid: 'wf-1', version: '1.0.0', last_modified_date: DATE, schemaVersion: '4.0',
    value_property_specifications: (opts.valueProps as never) ?? [
      { name: 'FailureContext', entries: [{ name: 'Mode', value: '' }, { name: 'Message', value: '' }] },
    ],
    steps: [
      s({ local_id: 'Start', oid: 's1', step_type: 'START' }),
      s({ local_id: 'Action', oid: 's2', step_type: 'ACTION PROXY',
          try_specifications: opts.trySpec ?? [{ mode: 'ERROR', catch_id: 'C1' }] }),
      s({ local_id: 'End', oid: 's3', step_type: 'END' }),
      s({ local_id: 'Catch', oid: 'c1', step_type: 'CATCH', catch_id: 'C1',
          output_parameter_specifications: opts.catchOutputs ?? [
            { id: 'trigger_reason', target: 'FailureContext.Mode' },
            { id: 'error_message', target: 'FailureContext.Message' },
          ] }),
      ...(opts.extraCatchSteps ?? []) as never[],
      s({ local_id: 'Ret', oid: 'r1', step_type: 'RETURN', return_config: opts.returnConfig ?? { command: 'ABANDON' } }),
    ] as never,
    connections: [
      { from_step_id: 's1', to_step_id: 's2' },
      { from_step_id: 's2', to_step_id: 's3' },
      ...(opts.extraCatchConns ?? [{ from_step_id: 'c1', to_step_id: 'r1' }]) as never[],
    ] as never,
    environment_specifications: [],
  } as MasterWorkflowSpecification;
}
function traceStates(engine: WorkflowEngine) {
  return engine.getTrace().map(t => `${t.step_oid}:${t.state}`);
}
```

### Task D1: `'fail'` signal — uncaught failure errors the workflow

This replaces today's silent-swallow even for workflows with no TRY: an uncaught action failure now properly errors the workflow.

**Files:** Modify `engines/web/src/engine.ts`; create `engines/web/src/try-catch-engine.test.ts`.

- [ ] **Step 1: Write the failing test** — create `try-catch-engine.test.ts` with the shared fixture above, then:
```typescript
describe('engine: fail signal (no matching TRY)', () => {
  it('errors the workflow when a failed action has no TRY', () => {
    const wf = tryWorkflow({ trySpec: [] }); // Action has no try_specifications
    const engine = new WorkflowEngine(wf);
    engine.start();
    engine.submitAction({ step_oid: 's2', action: 'fail', failure_mode: 'ERROR', error: 'boom' }, 0);
    assert.equal(engine.getWorkflowState(), 'ERRORED');
    assert.ok(traceStates(engine).includes('s2:ERRORED'));
  });
});
```

- [ ] **Step 2: Run — expect failure** (`npm run build && node --test dist/try-catch-engine.test.js`): `submitAction` throws or marks COMPLETED, not ERRORED.

- [ ] **Step 3: Implement** — in `engine.ts`:

Add the import (extend the existing `./types.js` type import, line 3-18):
```typescript
  CatchContext,
  FailureMode,
```
Add `activateCatchStep` to the step-handlers import (line 23):
```typescript
import { isAutoCompleting, needsUserAction, handleSelect1, handleUserAction, getFormElements, canonicalStepType, executeScript, activateCatchStep } from './step-handlers.js';
```
Add the import for the partition helper:
```typescript
import { partitionCatchNetworks, type CatchNetworkPartition } from './catch-network-partition.js';
```
Add a field (next to the other private fields, ~line 44):
```typescript
  private active_catches: Map<string, CatchContext> = new Map();
```
In `submitAction`, immediately after the `EXECUTING` guard (line 201), add the fail branch:
```typescript
    if (action.action === 'fail') {
      this.handleStepFailure(stepInstance, action.failure_mode ?? 'ERROR', action.error ?? null, _actionIndex);
      return;
    }
```
Add the method (anywhere among the private methods):
```typescript
  private handleStepFailure(stepInstance: StepInstance, mode: FailureMode, error: string | null, actionIndex: number): void {
    // (TRY routing added in Task D2.) Default: uncaught failure errors the workflow.
    this.recordTrace(stepInstance.oid, 'ERRORED', actionIndex, error ?? undefined);
    stepInstance.state = 'ERRORED';
    this.workflowState = 'ERRORED';
    this.pendingUserSteps.delete(stepInstance.oid);
    if (this.resourceManager) {
      const known = new Set<string>();
      this.collectKnownStepOids(known);
      this.resourceManager.cancelQueuedWaiters(known);
    }
  }
```
(This mirrors the existing SCRIPT-error path at `engine.ts:917-925`. `collectKnownStepOids` already exists and is used there.)

- [ ] **Step 4: Run — expect pass.**

- [ ] **Step 5: Commit**
```
git add engines/web/src/engine.ts engines/web/src/try-catch-engine.test.ts
git commit -m "feat(engine): fail signal — uncaught action failure errors the workflow"
```

### Task D2: TRY routing → CATCH activation

**Files:** Modify `engines/web/src/engine.ts`, `engines/web/src/try-catch-engine.test.ts`.

- [ ] **Step 1: Add failing test** — append:
```typescript
describe('engine: TRY routing to CATCH', () => {
  it('activates the CATCH network and writes trigger info on a matching failure', () => {
    const wf = tryWorkflow(); // Action TRY ERROR→C1; catch C1→Ret(ABANDON)
    const engine = new WorkflowEngine(wf);
    engine.start();
    engine.submitAction({ step_oid: 's2', action: 'fail', failure_mode: 'ERROR', error: 'thermocouple failure' }, 0);
    const states = traceStates(engine);
    assert.ok(states.includes('c1:COMPLETED'), `CATCH not activated: ${states.join(', ')}`);
    assert.ok(states.includes('r1:COMPLETED'), 'RETURN not reached');
    assert.equal(engine.getProperties()['FailureContext.Mode'], 'ERROR');
    assert.equal(engine.getProperties()['FailureContext.Message'], 'thermocouple failure');
    assert.notEqual(engine.getWorkflowState(), 'ERRORED'); // failure was caught
  });
});
```
(RETURN still auto-completes as a no-op here — actual ABANDON effect lands in Task E1.)

- [ ] **Step 2: Run — expect failure.**

- [ ] **Step 3: Implement** — in `engine.ts`:

In `activateStepAfterResources`, inside the `if (isAutoCompleting(target.stepType))` block, add a CATCH pre-step (before `this.recordTrace(target.oid, 'COMPLETED')` at line 943):
```typescript
      if (target.stepType === 'CATCH') {
        const ctx = this.active_catches.get(target.oid);
        if (ctx) activateCatchStep(target.step, ctx, this.propertyStore);
      }
```
Add the catch lookup + a real TRY-match path. Add these methods:
```typescript
  private findCatchByCatchId(catchId: string): StepInstance | undefined {
    for (const s of this.steps.values()) {
      if (s.stepType === 'CATCH' && s.step.catch_id === catchId) return s;
    }
    return undefined;
  }

  private buildPartition(): CatchNetworkPartition {
    const steps = [...this.steps.values()].map(s => ({ oid: s.oid, step_type: s.stepType, catch_id: s.step.catch_id }));
    const conns = this.connections.map(c => ({ from_step_id: c.from_step_id, to_step_id: c.to_step_id }));
    return partitionCatchNetworks(steps, conns);
  }
```
Replace the body of `handleStepFailure` with the match-then-fallback version:
```typescript
  private handleStepFailure(stepInstance: StepInstance, mode: FailureMode, error: string | null, actionIndex: number): void {
    const matching = stepInstance.step.try_specifications?.find(t => t.mode === mode);
    const catchStep = matching ? this.findCatchByCatchId(matching.catch_id) : undefined;

    if (matching && catchStep) {
      if (this.active_catches.has(catchStep.oid)) {
        // CATCH_REENTRY (spec §6.4): the catch is already active → abort the workflow.
        this.recordTrace(stepInstance.oid, 'ERRORED', actionIndex, `CATCH_REENTRY on ${matching.catch_id}`);
        stepInstance.state = 'ERRORED';
        this.workflowState = 'ABORTED';
        return;
      }
      this.active_catches.set(catchStep.oid, {
        catch_oid: catchStep.oid,
        trigger_step_oid: stepInstance.oid,
        trigger_step_name: stepInstance.step.local_id,
        trigger_reason: mode,
        error_message: error,
        activated_at: new Date().toISOString(),
      });
      // Deactivate the trigger step (caught — do NOT propagate failure).
      this.recordTrace(stepInstance.oid, 'IDLE', actionIndex);
      stepInstance.state = 'IDLE';
      this.pendingUserSteps.delete(stepInstance.oid);
      // Run the catch network.
      this.activateStep(catchStep);
      this.drainCompletionQueue();
      return;
    }

    // No matching TRY — uncaught failure errors the workflow (mirrors SCRIPT-error path).
    this.recordTrace(stepInstance.oid, 'ERRORED', actionIndex, error ?? undefined);
    stepInstance.state = 'ERRORED';
    this.workflowState = 'ERRORED';
    this.pendingUserSteps.delete(stepInstance.oid);
    if (this.resourceManager) {
      const known = new Set<string>();
      this.collectKnownStepOids(known);
      this.resourceManager.cancelQueuedWaiters(known);
    }
  }
```

- [ ] **Step 4: Run — expect pass** (D1's no-TRY test still passes via the fallback).

- [ ] **Step 5: Commit**
```
git add engines/web/src/engine.ts engines/web/src/try-catch-engine.test.ts
git commit -m "feat(engine): TRY routing activates CATCH network with trigger context"
```

---

## Phase E — RETURN command dispatch

RETURN is intercepted in the engine **before** the generic auto-complete path so its redirective/terminal effects are final (it must not fall through and re-push itself).

### Task E1: RETURN dispatch scaffold + ABANDON

**Files:** Modify `engines/web/src/engine.ts`, `engines/web/src/try-catch-engine.test.ts`.

- [ ] **Step 1: Add failing test** — append:
```typescript
describe('engine: RETURN ABANDON', () => {
  it('aborts the workflow on RETURN ABANDON', () => {
    const engine = new WorkflowEngine(tryWorkflow({ returnConfig: { command: 'ABANDON' } }));
    engine.start();
    engine.submitAction({ step_oid: 's2', action: 'fail', failure_mode: 'ERROR', error: 'x' }, 0);
    assert.equal(engine.getWorkflowState(), 'ABORTED');
    assert.equal(engine.active_catchesSize?.() ?? 0, 0); // catch context cleaned up (helper below)
  });
});
```
(If you don't want a test-only accessor, drop the second assert; the ABORTED state is the key behaviour.)

- [ ] **Step 2: Run — expect failure** (workflow ends RUNNING/COMPLETED, not ABORTED, because RETURN currently no-ops).

- [ ] **Step 3: Implement** — in `engine.ts`:

In `activateStepAfterResources`, add a RETURN intercept **before** the `if (isAutoCompleting(...))` block (right after the WORKFLOW PROXY block, ~line 904):
```typescript
    if (target.stepType === 'RETURN') {
      this.recordTrace(target.oid, 'COMPLETED');
      target.state = 'COMPLETED';
      this.dispatchReturn(target);
      return; // RETURN's command is terminal/redirective — do not fall through.
    }
```
Add the dispatcher + ABANDON (and the cleanup helper):
```typescript
  private dispatchReturn(returnStep: StepInstance): void {
    const rc = returnStep.step.return_config;
    if (!rc) return;
    const catchOid = this.findActiveCatchForReturn(returnStep.oid);
    const ctx = catchOid ? this.active_catches.get(catchOid) : undefined;
    switch (rc.command) {
      case 'ABANDON': this.returnAbandon(); break;
      case 'RESTART': this.returnRestart(rc.restart_mode ?? 'KEEP'); break;       // Task E2
      case 'GOTO': if (rc.goto_step_oid) this.returnGoto(rc.goto_step_oid); break; // Task E3
      case 'RETRY': this.returnRetry(ctx); break;                                  // Task E4
    }
    if (catchOid) {
      this.cleanupCatchNetwork(catchOid);
      this.active_catches.delete(catchOid);
    }
  }

  private findActiveCatchForReturn(returnOid: string): string | undefined {
    const partition = this.buildPartition();
    for (const [catchId, net] of partition.networksByCatchId) {
      if (net.has(returnOid)) {
        const catchStep = this.findCatchByCatchId(catchId);
        if (catchStep && this.active_catches.has(catchStep.oid)) return catchStep.oid;
      }
    }
    return undefined;
  }

  private cleanupCatchNetwork(catchOid: string): void {
    const catchStep = this.steps.get(catchOid);
    const cid = catchStep?.step.catch_id;
    const net = cid ? this.buildPartition().networksByCatchId.get(cid) : undefined;
    for (const oid of net ?? []) {
      const st = this.steps.get(oid);
      if (st && st.state !== 'IDLE') { this.recordTrace(oid, 'IDLE'); st.state = 'IDLE'; }
    }
  }

  private returnAbandon(): void {
    for (const step of this.steps.values()) {
      if (ACTIVE_STEP_STATES.has(step.state)) { this.recordTrace(step.oid, 'IDLE'); step.state = 'IDLE'; }
    }
    this.completionQueue.length = 0; // cancel any queued activations
    if (this.resourceManager) this.releaseAllResources();
    this.workflowState = 'ABORTED';
  }
```
(Optional test accessor — add if you used the second assert: `active_catchesSize(): number { return this.active_catches.size; }`. Stub `returnRestart`/`returnGoto`/`returnRetry` as empty methods for now; they're implemented next.)

- [ ] **Step 4: Run — expect pass.**

- [ ] **Step 5: Commit**
```
git add engines/web/src/engine.ts engines/web/src/try-catch-engine.test.ts
git commit -m "feat(engine): RETURN dispatch scaffold + ABANDON (workflow → ABORTED)"
```

### Task E2: RESTART (CLEAN and KEEP)

**Files:** Modify `engine.ts`, `try-catch-engine.test.ts`.

- [ ] **Step 1: Add failing tests** — append:
```typescript
describe('engine: RETURN RESTART', () => {
  it('RESTART KEEP re-runs from START and preserves properties', () => {
    const wf = tryWorkflow({
      returnConfig: { command: 'RESTART', restart_mode: 'KEEP' },
      catchOutputs: [{ id: 'trigger_reason', target: 'FailureContext.Mode' }],
    });
    const engine = new WorkflowEngine(wf);
    engine.start();
    engine.submitAction({ step_oid: 's2', action: 'fail', failure_mode: 'ERROR', error: 'x' }, 0);
    assert.equal(engine.getWorkflowState(), 'RUNNING');
    assert.equal(engine.getProperties()['FailureContext.Mode'], 'ERROR'); // preserved
    // Action s2 is active again after restart (re-run from START).
    assert.ok(engine.getActiveSteps().some(a => a.step.oid === 's2'));
  });

  it('RESTART CLEAN resets properties to defaults', () => {
    const wf = tryWorkflow({ returnConfig: { command: 'RESTART', restart_mode: 'CLEAN' } });
    const engine = new WorkflowEngine(wf);
    engine.start();
    engine.submitAction({ step_oid: 's2', action: 'fail', failure_mode: 'ERROR', error: 'x' }, 0);
    assert.equal(engine.getProperties()['FailureContext.Mode'], ''); // reset to declared default
  });
});
```
(Confirm `getActiveSteps()` returns entries with `.step.oid`; the recon noted `getActiveSteps()` exists at engine.ts:324 — adjust the accessor if its shape differs.)

- [ ] **Step 2: Run — expect failure.**

- [ ] **Step 3: Implement** — replace the `returnRestart` stub:
```typescript
  private returnRestart(mode: 'CLEAN' | 'KEEP'): void {
    for (const step of this.steps.values()) {
      if (step.state !== 'IDLE') { this.recordTrace(step.oid, 'IDLE'); step.state = 'IDLE'; }
    }
    this.completionQueue.length = 0;
    this.pendingUserSteps.clear();
    this.active_catches.clear();
    if (this.resourceManager) this.releaseAllResources();
    if (mode === 'CLEAN') {
      // Re-initialise Value Properties to their declared defaults (same call the ctor uses).
      this.propertyStore.initializeFromWorkflow(this.workflow);
    }
    // Re-fire START (mirror start()'s START handling).
    const startStep = [...this.steps.values()].find(st => st.stepType === 'START');
    if (startStep) {
      this.recordTrace(startStep.oid, 'COMPLETED');
      startStep.state = 'COMPLETED';
      this.completionQueue.push(startStep.oid);
    }
  }
```
Note: `dispatchReturn` calls `cleanupCatchNetwork` + `active_catches.delete` AFTER `returnRestart`; since RESTART already cleared `active_catches` and idled every step, those are harmless no-ops. The `drainCompletionQueue` loop that invoked this RETURN continues and drains the freshly-pushed START, re-running the main flow.

- [ ] **Step 4: Run — expect pass.**

- [ ] **Step 5: Commit**
```
git add engines/web/src/engine.ts engines/web/src/try-catch-engine.test.ts
git commit -m "feat(engine): RETURN RESTART (CLEAN re-inits properties; KEEP preserves)"
```

### Task E3: GOTO

**Files:** Modify `engine.ts`, `try-catch-engine.test.ts`.

- [ ] **Step 1: Add failing test** — append:
```typescript
describe('engine: RETURN GOTO', () => {
  it('resumes the main flow at the GOTO target', () => {
    // main flow: Start→Action→Mid→End ; GOTO target = Mid (a USER_INTERACTION)
    const wf = tryWorkflow({
      returnConfig: { command: 'GOTO', goto_step_oid: 'm1' },
      extraCatchSteps: [s({ local_id: 'Mid', oid: 'm1', step_type: 'USER_INTERACTION' })],
      extraCatchConns: [{ from_step_id: 'c1', to_step_id: 'r1' }],
    });
    // rewire main flow to include Mid: Start→Action→Mid→End
    (wf.connections as Array<Record<string, string>>).length = 0;
    (wf.connections as Array<Record<string, string>>).push(
      { from_step_id: 's1', to_step_id: 's2' },
      { from_step_id: 's2', to_step_id: 'm1' },
      { from_step_id: 'm1', to_step_id: 's3' },
      { from_step_id: 'c1', to_step_id: 'r1' },
    );
    const engine = new WorkflowEngine(wf);
    engine.start();
    engine.submitAction({ step_oid: 's2', action: 'fail', failure_mode: 'ERROR', error: 'x' }, 0);
    assert.ok(engine.getActiveSteps().some(a => a.step.oid === 'm1'), 'GOTO target not active');
    assert.notEqual(engine.getWorkflowState(), 'ERRORED');
  });
});
```

- [ ] **Step 2: Run — expect failure.**

- [ ] **Step 3: Implement** — replace the `returnGoto` stub:
```typescript
  private returnGoto(gotoOid: string): void {
    const target = this.steps.get(gotoOid);
    if (!target) return;
    if (target.state !== 'IDLE') this.resetStep(gotoOid);
    this.activateStep(target); // branch-local: other active branches untouched
  }
```

- [ ] **Step 4: Run — expect pass.** **Step 5: Commit**
```
git add engines/web/src/engine.ts engines/web/src/try-catch-engine.test.ts
git commit -m "feat(engine): RETURN GOTO resumes main flow at target"
```

### Task E4: RETRY

**Files:** Modify `engine.ts`, `try-catch-engine.test.ts`.

- [ ] **Step 1: Add failing test** — append:
```typescript
describe('engine: RETURN RETRY', () => {
  it('re-invokes the trigger action; second attempt can succeed', () => {
    const engine = new WorkflowEngine(tryWorkflow({ returnConfig: { command: 'RETRY' } }));
    engine.start();
    engine.submitAction({ step_oid: 's2', action: 'fail', failure_mode: 'ERROR', error: 'x' }, 0);
    // After RETRY, s2 is EXECUTING again — supply a successful completion.
    assert.ok(engine.getActiveSteps().some(a => a.step.oid === 's2'), 's2 not re-activated');
    engine.submitAction({ step_oid: 's2', action: 'submit', form_values: {} }, 1);
    assert.equal(engine.getWorkflowState(), 'COMPLETED');
  });
});
```

- [ ] **Step 2: Run — expect failure.**

- [ ] **Step 3: Implement** — replace the `returnRetry` stub:
```typescript
  private returnRetry(ctx?: CatchContext): void {
    if (!ctx) return;
    const trigger = this.steps.get(ctx.trigger_step_oid);
    if (!trigger) return;
    if (trigger.state !== 'IDLE') this.resetStep(ctx.trigger_step_oid);
    this.activateStep(trigger); // ACTION PROXY → EXECUTING again
  }
```
(Per the descoped resource decision, RETRY re-invokes against whatever resource state exists; the workflow author manages re-acquire via the step's own Acquire commands.)

- [ ] **Step 4: Run — expect pass.** **Step 5: Commit**
```
git add engines/web/src/engine.ts engines/web/src/try-catch-engine.test.ts
git commit -m "feat(engine): RETURN RETRY re-invokes the trigger action"
```

---

## Phase F — Production wiring (`web-ui` coordinator)

Run these tests from `engines/web-ui`: `node --import tsx --test src/<path>/<name>.test.ts`.

### Task F1: Classify the terminal into ERROR / ABORT / TIMEOUT

**Files:**
- Modify: `engines/web-ui/src/actionProxy/stateMapping.ts`
- Modify: `engines/web-ui/src/actionProxy/ActionProxyController.ts` (`ControllerConfig.onTerminal`, `emitTerminal`)
- Modify: `engines/web-ui/src/coordinator/WorkflowCoordinator.ts` (`startActionProxy` onTerminal param)
- Modify: `engines/web-ui/src/actionProxy/ActionProxyController.test.ts`

- [ ] **Step 1: Add failing test** — append to `ActionProxyController.test.ts` (mirror the existing `'on ABORTED signals terminal as ERRORED'` test at lines 74-92):
```typescript
test('classifies ABORTED terminal as failureMode "abort"', async () => {
  const observer = new FakeObserver();
  const captured: CapturedTerminal[] = [];
  const controller = new ActionProxyController({
    serverUri: 'http://localhost:3002', actionOid: 'act-1',
    invokeRequest: { environment_oid: 'env-1', workflow_instance_id: 'wf-1', step_instance_id: 'si-1', step_oid: 'step-1', input_parameters: [] },
    visibility: 'observable', supportedCommands: ['ABORT'],
    persistence: new PersistenceStore(new FakeStorage()), observer,
    fetchImpl: fakeFetch({ status: 201, body: { data: { instance_id: 'ai-1' }, meta: {} } }) as typeof fetch,
    onTerminal: t => captured.push(t),
  });
  await controller.start();
  observer.emit({ kind: 'state_change', state: 'ABORTED', previous_state: 'RUNNING', ts: 't', eventId: 1 });
  assert.equal(captured[0].state, 'ERRORED');
  assert.equal(captured[0].failureMode, 'abort');
});
```
(Extend the test file's `CapturedTerminal` type with `failureMode?: 'error' | 'abort' | 'timeout' | null`.)

- [ ] **Step 2: Run — expect failure** (`node --import tsx --test src/actionProxy/ActionProxyController.test.ts`): `failureMode` is undefined.

- [ ] **Step 3: Implement**

In `stateMapping.ts`, add (do NOT modify the existing `mapServerStateToEngineState` — it has live tests + another consumer):
```typescript
export function mapServerStateToFailureMode(s: ServerState): 'error' | 'abort' | null {
  switch (s) {
    case 'ABORTED':
    case 'STOPPED':
      return 'abort';
    case 'ERRORED':
      return 'error';
    default:
      return null;
  }
}
```
In `ActionProxyController.ts`, widen `ControllerConfig.onTerminal` (line 25):
```typescript
  onTerminal: (t: { state: 'COMPLETED' | 'ERRORED'; outputs: Record<string, string>; errorMessage: string | null; failureMode: 'error' | 'abort' | 'timeout' | null }) => void;
```
Add the import and set `failureMode` in `emitTerminal` (lines 179-183). Give `emitTerminal` an optional explicit override (used by the timeout path in F3):
```typescript
  private emitTerminal(state: ServerState, errorMessage: string | null, failureModeOverride?: 'timeout'): void {
    if (this.terminalEmitted) return;
    this.terminalEmitted = true;
    this.clearTimeout(); // added in F3 (no-op if not present)
    this.dispose();
    const engineState = mapServerStateToEngineState(state);
    this.cfg.persistence.remove(this.cfg.invokeRequest.step_instance_id);
    this.cfg.persistence.flushSync();
    this.cfg.onTerminal({
      state: engineState === 'COMPLETED' ? 'COMPLETED' : 'ERRORED',
      outputs: Object.fromEntries(this.outputs),
      errorMessage,
      failureMode: engineState === 'COMPLETED' ? null : (failureModeOverride ?? mapServerStateToFailureMode(state)),
    });
  }
```
(Import `mapServerStateToFailureMode` from `./stateMapping.js`. If you haven't done F3 yet, replace `this.clearTimeout();` with nothing.)

In `WorkflowCoordinator.ts`, widen the `onTerminal` param type of `startActionProxy` (line 193) and the synthetic invoke-failure emit (line 221) to include `failureMode`:
```typescript
    onTerminal: (t: { state: 'COMPLETED' | 'ERRORED'; outputs: Record<string, string>; errorMessage: string | null; failureMode: 'error' | 'abort' | 'timeout' | null }) => void,
```
and at line 221:
```typescript
      onTerminal({ state: 'ERRORED', outputs: {}, errorMessage: String(e instanceof Error ? e.message : e), failureMode: 'error' });
```

- [ ] **Step 4: Run — expect pass** (also re-run the existing controller tests: `node --import tsx --test src/actionProxy/ActionProxyController.test.ts`).

- [ ] **Step 5: Commit**
```
git add engines/web-ui/src/actionProxy/stateMapping.ts engines/web-ui/src/actionProxy/ActionProxyController.ts engines/web-ui/src/coordinator/WorkflowCoordinator.ts engines/web-ui/src/actionProxy/ActionProxyController.test.ts
git commit -m "feat(web-ui): classify action terminal into error/abort/timeout"
```

### Task F2: Feed the fail signal into the engine

**Files:** Modify `engines/web-ui/src/coordinator/WorkflowCoordinator.ts` (the `onTerminal` callback in `sync()`, lines 477-499).

- [ ] **Step 1:** Replace the empty-submit stub (the `else` branch at lines 486-499) with a real fail signal:
```typescript
        } else {
          // Real per-step failure: route it into the engine's TRY/CATCH machinery.
          console.warn(`[ActionProxy] step ${step.oid} terminated as ERRORED (${t.failureMode}): ${t.errorMessage}`);
          try {
            this.engine.submitAction({
              step_oid: step.oid,
              action: 'fail',
              failure_mode: (t.failureMode === 'abort' ? 'ABORT' : t.failureMode === 'timeout' ? 'TIMEOUT' : 'ERROR'),
              error: t.errorMessage ?? undefined,
            }, this.actionIndex++);
            this.sync();
          } catch (e) {
            this.publish({ ...this.snapshot, error: String(e) });
          }
        }
```
(The `UserAction` `'fail'` variant + `failure_mode`/`error` fields flow from `@engine/types.js` — already added in engine Task C1. `FailureMode` is the engine's `'ERROR'|'ABORT'|'TIMEOUT'`.)

- [ ] **Step 2: Verify** — `cd engines/web-ui && npm test` (this runs `tsc -b` then the suite; the type-check confirms the `'fail'` action is accepted and existing tests still pass). This wiring has no isolable unit (it needs a live engine + container); it is exercised end-to-end by `test/integration` and manual runs. Confirm a clean `tsc -b`.

- [ ] **Step 3: Commit**
```
git add engines/web-ui/src/coordinator/WorkflowCoordinator.ts
git commit -m "feat(web-ui): feed real action failures into the engine fail signal"
```

### Task F3: Runtime wall-clock timeout

**Files:** Modify `engines/web-ui/src/actionProxy/ActionProxyController.ts` (+ `ControllerConfig`), `WorkflowCoordinator.ts` (pass `timeoutMs`), `ActionProxyController.test.ts`.

- [ ] **Step 1: Add failing test** — append to `ActionProxyController.test.ts`, injecting a controllable timer:
```typescript
test('fires ABORT and a timeout terminal when the wall-clock elapses', async () => {
  const observer = new FakeObserver();
  const captured: CapturedTerminal[] = [];
  let fire: (() => void) | null = null;
  const controller = new ActionProxyController({
    serverUri: 'http://localhost:3002', actionOid: 'act-1',
    invokeRequest: { environment_oid: 'env-1', workflow_instance_id: 'wf-1', step_instance_id: 'si-1', step_oid: 'step-1', input_parameters: [] },
    visibility: 'observable', supportedCommands: ['ABORT'],
    persistence: new PersistenceStore(new FakeStorage()), observer,
    fetchImpl: fakeFetch({ status: 201, body: { data: { instance_id: 'ai-1' }, meta: {} } }) as typeof fetch,
    timeoutMs: 1000,
    setTimeoutImpl: (cb: () => void) => { fire = cb; return 1 as unknown as ReturnType<typeof setTimeout>; },
    clearTimeoutImpl: () => { fire = null; },
    onTerminal: t => captured.push(t),
  });
  await controller.start();
  assert.ok(fire, 'timer not scheduled');
  fire!();
  assert.equal(captured.length, 1);
  assert.equal(captured[0].state, 'ERRORED');
  assert.equal(captured[0].failureMode, 'timeout');
});
```

- [ ] **Step 2: Run — expect failure.**

- [ ] **Step 3: Implement** — in `ActionProxyController.ts`:

Add to `ControllerConfig` (lines 16-26):
```typescript
  timeoutMs?: number;
  setTimeoutImpl?: (cb: () => void, ms: number) => ReturnType<typeof setTimeout>;
  clearTimeoutImpl?: (h: ReturnType<typeof setTimeout>) => void;
```
Add a private field `private timeoutHandle: ReturnType<typeof setTimeout> | null = null;` and the helper:
```typescript
  private clearTimeout(): void {
    if (this.timeoutHandle !== null) {
      (this.cfg.clearTimeoutImpl ?? clearTimeout)(this.timeoutHandle);
      this.timeoutHandle = null;
    }
  }
```
In `start()`, right after `this.setSnapshot({ ...this.snapshot, instanceId: instance_id, serverState: 'IDLE' });` (line 82):
```typescript
    if (this.cfg.timeoutMs && this.cfg.timeoutMs > 0) {
      const set = this.cfg.setTimeoutImpl ?? setTimeout;
      this.timeoutHandle = set(() => {
        void this.api.sendCommand(this.cfg.serverUri, instance_id, 'ABORT').catch(() => { /* best effort */ });
        this.emitTerminal('ERRORED', 'Action timed out', 'timeout');
      }, this.cfg.timeoutMs);
    }
```
Ensure `clearTimeout()` is called in `dispose()` (lines 128-133) and at the top of `emitTerminal` (already added in F1's `emitTerminal`).

In `WorkflowCoordinator.ts` `startActionProxy` (the `new ActionProxyController({...})` at lines 199-217), pass the timeout sourced from the step's `action_proxy_config.timeout_ms` (already on the type at `types.ts:345`):
```typescript
        timeoutMs: cap.visibility === 'observable' ? (this._actionTimeoutMs ?? undefined) : undefined,
```
(Add a `timeoutMs` derivation appropriate to your config source — e.g. read `step.step.action_proxy_config?.timeout_ms` where `sync()` calls `startActionProxy`. If no per-action timeout is configured, leave it undefined and the timer is simply not scheduled.)

- [ ] **Step 4: Run — expect pass** (and re-run the full controller test file).

- [ ] **Step 5: Commit**
```
git add engines/web-ui/src/actionProxy/ActionProxyController.ts engines/web-ui/src/coordinator/WorkflowCoordinator.ts engines/web-ui/src/actionProxy/ActionProxyController.test.ts
git commit -m "feat(web-ui): runtime wall-clock action timeout → ABORT + timeout terminal"
```

---

## Self-Review (run after completing all tasks)

- [ ] **Full TS suite green:** `cd engines/web && npm test` (all `node:test` files incl. the existing suite — no regressions in `validator.test.ts`, `engine-*.test.ts`).
- [ ] **web-ui suite green:** `cd engines/web-ui && npm test` (`tsc -b` clean + tests).
- [ ] **Existing conformance unaffected:** `cd engines/web && npm run conformance` (we added no fixtures; the existing suite must still pass — confirms the validator changes didn't break existing validation/execution fixtures, e.g. `exec-action-proxy-001`).
- [ ] **Spec coverage check (§6 error codes):** every code in spec Appendix A that is marked "structural/cross-reference/topology" maps to a B3–B5 test. `INVALID_TRY_MODE`/`INVALID_RESTART_MODE`/`INVALID_RETURN_COMMAND` are covered by tryCatchValidation; confirm each has a test (add any missing).
- [ ] **Spec coverage check (§5 commands):** ABANDON (E1), RESTART CLEAN+KEEP (E2), GOTO (E3), RETRY (E4) each have a passing engine test.
- [ ] **The silent-swallow bug is fixed:** D1's test proves an uncaught failure now ERRORs the workflow (was: silently COMPLETED).
- [ ] **No placeholders:** confirm `returnRestart`/`returnGoto`/`returnRetry` stubs from E1 were all replaced.
- [ ] **Deviations noted:** `return-dispatcher.ts` was folded into `engine.ts` private methods (RETURN dispatch is tightly coupled to engine state); `RETURN_WITHOUT_CATCH`/`CATCH_NETWORK_NOT_CONNECTED` are covered transitively (B5 note). `release_on_catch` is parsed/validated but not acted on (descoped). If a future fixture needs the explicit codes or auto-release, revisit.

## Execution Handoff

This plan is verified entirely by `node:test` unit tests. The **shared `spec/conformance` fixtures (exec-try-catch-001..010, valid V1..V7) are added in the sibling Kotlin plan** as the cross-engine parity capstone, so they prove both engines at once and neither suite is left red.

After this plan: execute `2026-05-31-try-catch-runtime-native-kmp.md` (Kotlin parity + the shared conformance fixtures). The editor and docs plans remain separate and should each get the same grounding check before execution.

