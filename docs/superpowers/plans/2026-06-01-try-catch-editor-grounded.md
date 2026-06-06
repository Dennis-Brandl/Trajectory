# TRY / CATCH / RETURN — Editor Implementation Plan (GROUNDED 2026-06-01)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Supersedes `2026-05-31-try-catch-editor.md`.** That plan was authored before the TrajectoryEditor repo was inspected and was misgrounded in 6 places. This version was validated against the real code (commit `27bd133`). The 6 corrections folded in:
> 1. **Store:** the live canvas graph lives in `src/stores/flowStore.ts` (`useFlowStore`, flat `nodes`/`edges`, zundo undo), NOT a flat `src/store/specStore.ts`. `specificationStore` holds the persisted, *nested* `spec.data.nodes`. The `renameCatchId` action goes in **flowStore**.
> 2. **`isValidConnection`:** it is an inline `useCallback` in `src/components/canvas/FlowCanvas.tsx` matching React Flow's real API — **one** arg (`Connection`), nodes/edges read from the store closure — NOT a standalone `(conn, graph)` util. We extend it via a pure helper.
> 3. **Validator:** the editor validator (`src/lib/export/workflow-validation.ts`) is **message-based** (`ValidationError[]` with `{nodeId,nodeName,nodeType,message}`), NOT error-code based, and there is **no importable runtime validator** (no `@trajectory/runtime` workspace package). We mirror the rules as editor messages and re-implement the partition helper locally.
> 4. **Renderers:** node components are **unified** (one component per type adapting via `useNotationConfig()`) in `src/components/nodes/renderers/unified/`, registered into all three notation maps — NOT per-notation files. We add ONE `CatchNode` + ONE `ReturnNode`. Styling is **Tailwind + inline SVG**, NOT CSS modules.
> 5. **`notation-config.ts`** has no per-node-type dimension map; trapezoid geometry lives in the component.
> 6. **Exporter/importer signatures:** `exportWorkflow(spec, nodes, edges, …)` and `importWorkflow(json): WorkflowImportSuccess | ImportError` — the round-trip test must use the real signatures.

**Goal:** Add the editor side of TRY/CATCH/RETURN — node types, palette, unified node renderers, property-panel editors, exporter/importer round-trip, message-based validation, and `catch_id` rename propagation — per `docs/superpowers/specs/2026-05-31-try-catch-return-design.md`.

**Architecture:** React Flow (`@xyflow/react` v12) + Zustand v5. The canvas graph is the flat `nodes`/`edges` in `useFlowStore`; it write-throughs to `useSpecificationStore` (which stores the nested `spec.data.nodes`). New node types are unified components that adapt per notation via `useNotationConfig()`. The TRY editor is a new PropertyPanel section mirroring `ResourceCommandEditor`. Validation is message-based in `workflow-validation.ts`; the catch-network partition helper is re-implemented locally (the runtime is not an importable dependency).

**Tech Stack:** TypeScript, React 19, `@xyflow/react` v12, Zustand v5 (+ zundo temporal, persist), Vitest + `@testing-library/react` + jsdom, Tailwind, AJV (`ajv` v8, browser). Editor camelCase (`trySpecifications`, `catchId`, `returnConfig`) ↔ wire snake_case (`try_specifications`, `catch_id`, `return_config`); export/import translate.

**Depends on:** the shared `spec/workflow-schema.json` already allows `CATCH`/`RETURN` step types + the new fields (landed during the runtime work). Verify before Phase M that the schema validator (`src/lib/import/schema-validator.ts` → AJV) accepts a workflow carrying `try_specifications`/`catch_id`/`return_config`; if it rejects them, add them to the editor's bundled schema first (a prerequisite micro-task at the top of Phase M).

**Out of scope:** runtime engine behavior + conformance fixtures (shipped), docs (sibling plan), android-app coordinator wiring (sibling follow-up).

**Test command:** `cd C:/Trajectory/TrajectoryEditor && npm run test:run -- <pattern>` (watch mode is `npm test`). Component tests need jsdom (already the Vitest default env here).

---

## File Structure Plan

### Files to modify
| File | Change |
|---|---|
| `src/types/nodes.ts` | Add `'catch' \| 'return'` to `NodeType`; add `trySpecifications?`, `catchId?`, `returnConfig?` to `StepNodeData`. |
| `src/lib/packageFormat/export-types.ts` | Add `catch: 'CATCH'`, `return: 'RETURN'` to `NODE_TYPE_TO_STEP_TYPE` (and `'CATCH'`/`'RETURN'` to the `StepType` union if it is a closed union). |
| `src/lib/import/type-mapping.ts` | Add `CATCH: 'catch'`, `RETURN: 'return'` to `STEP_TYPE_TO_NODE_TYPE` (the reverse map the importer uses). |
| `src/components/nodes/renderers/flowchart/index.ts` | Register `catch: CatchNode`, `return: ReturnNode`. |
| `src/components/nodes/renderers/bpmn/index.ts` | Same. |
| `src/components/nodes/renderers/isa88/index.ts` | Same. |
| `src/components/palette/StepPalette.tsx` | Add `catch`/`return` to `stepTypes` and `allStepTypes`. |
| `src/components/properties/NodeInlineEditor.tsx` | Add `'catch'` to `PARAMETERIZED_TYPES`, `'return'` to `CONTROL_FLOW_TYPES`; render `TryEditor` (action steps), `CatchIdEditor` (catch), `ReturnConfigEditor` (return). |
| `src/lib/packageFormat/step-extractors.ts` | Add `extractTrySpecifications`, `extractReturnConfig`, `extractCatchId`. |
| `src/lib/packageFormat/workflow-exporters.ts` | Emit `try_specifications`/`catch_id`/`return_config` in the per-node `nodes.map(...)` step object. |
| `src/lib/import/workflow-importers.ts` | In `transformWorkflowSpec`'s per-step `nodeData`, read `try_specifications`/`catch_id`/`return_config` into `trySpecifications`/`catchId`/`returnConfig`. |
| `src/components/canvas/FlowCanvas.tsx` | Extend the existing `isValidConnection` callback to reject CATCH-incoming, RETURN-outgoing, and cross-network edges. |
| `src/lib/export/workflow-validation.ts` | Add `catch`/`return` to `NODE_RULES`; add `validateTryCatchReturn(nodes, edges)` and call it from `validateWorkflow`. |
| `src/stores/flowStore.ts` + `src/stores/types.ts` | Add a `renameCatchId(oldId, newId)` action (and its `FlowState` type member). |

### Files to create
| File | Why |
|---|---|
| `src/components/nodes/renderers/unified/CatchNode.tsx` | Unified CATCH renderer (wide-top trapezoid; Tailwind + inline SVG; adapts via `useNotationConfig`). Default export. |
| `src/components/nodes/renderers/unified/ReturnNode.tsx` | Unified RETURN renderer (narrow-top trapezoid). Default export. |
| `src/components/properties/TryEditor.tsx` | TRY section in PropertyPanel (after Resource Commands on action steps). |
| `src/components/properties/ReturnConfigEditor.tsx` | RETURN command + conditional sub-field editor. |
| `src/components/properties/CatchIdEditor.tsx` | CATCH `catch_id` input with live uniqueness check. |
| `src/lib/canvas/catch-network-partition.ts` | Local port of the runtime's `partitionCatchNetworks` (BFS-from-CATCH), adapted to editor nodes/edges. |
| `src/lib/canvas/try-catch-connection.ts` | Pure `isTryCatchConnectionValid(connection, nodes, edges)` helper used by FlowCanvas's `isValidConnection`. |
| `src/components/properties/__tests__/TryEditor.test.tsx` | RTL component tests. |
| `src/components/properties/__tests__/ReturnConfigEditor.test.tsx` | RTL component tests. |
| `src/lib/canvas/__tests__/catch-network-partition.test.ts` | Partition unit tests. |
| `src/lib/canvas/__tests__/try-catch-connection.test.ts` | Connection-rule unit tests. |
| `src/lib/packageFormat/__tests__/try-catch-roundtrip.test.ts` | Export → import round-trip (real signatures). |
| `src/stores/__tests__/catch-rename-propagation.test.ts` | flowStore `renameCatchId` propagation. |
| `src/lib/export/__tests__/try-catch-validation.test.ts` | Message-based validation rules. |

---

## Phase I — Types & mappings

### Task I1: NodeType union, StepNodeData fields, and both step-type maps

**Files:** Modify `src/types/nodes.ts`, `src/lib/packageFormat/export-types.ts`, `src/lib/import/type-mapping.ts`. Test: `src/lib/packageFormat/__tests__/node-type-mapping.test.ts` (create).

- [ ] **Step 1: Failing test** — `src/lib/packageFormat/__tests__/node-type-mapping.test.ts`:
```typescript
import { describe, it, expect } from 'vitest'
import { NODE_TYPE_TO_STEP_TYPE } from '../export-types'
import { STEP_TYPE_TO_NODE_TYPE } from '../../import/type-mapping'

describe('TRY/CATCH/RETURN type mappings', () => {
  it('maps catch ↔ CATCH and return ↔ RETURN', () => {
    expect(NODE_TYPE_TO_STEP_TYPE.catch).toBe('CATCH')
    expect(NODE_TYPE_TO_STEP_TYPE.return).toBe('RETURN')
    expect(STEP_TYPE_TO_NODE_TYPE.CATCH).toBe('catch')
    expect(STEP_TYPE_TO_NODE_TYPE.RETURN).toBe('return')
  })
})
```
- [ ] **Step 2: Run — expect fail.** `npm run test:run -- node-type-mapping`
- [ ] **Step 3: Implement.**
  - `src/types/nodes.ts` — extend the union (keep the existing 11 members; append):
```typescript
export type NodeType =
  | 'start' | 'end'
  | 'actionProxyWait' | 'yesNoStep' | 'userInteractionStep' | 'scriptStep'
  | 'workflowProxy' | 'select1' | 'waitAny' | 'parallel' | 'waitAll'
  | 'catch' | 'return'
```
  - `src/types/nodes.ts` — add to `StepNodeData` (after `actionProxyConfig`):
```typescript
  /** TRY specifications on ACTION PROXY steps (camelCase editor side; exported as try_specifications). */
  trySpecifications?: Array<{ mode: 'ERROR' | 'ABORT' | 'TIMEOUT'; catch_id: string; release_on_catch?: boolean }>
  /** CATCH step's catch_id (the handle TRY entries reference). */
  catchId?: string
  /** RETURN step's command config (exported as return_config). */
  returnConfig?: { command: 'ABANDON' | 'RESTART' | 'GOTO' | 'RETRY'; restart_mode?: 'CLEAN' | 'KEEP'; goto_step_oid?: string }
```
  - `src/lib/packageFormat/export-types.ts` — add to `NODE_TYPE_TO_STEP_TYPE` (and add `'CATCH'`/`'RETURN'` to the `StepType` union if it is closed):
```typescript
  catch: 'CATCH',
  return: 'RETURN',
```
  - `src/lib/import/type-mapping.ts` — add to `STEP_TYPE_TO_NODE_TYPE`:
```typescript
  CATCH: 'catch',
  RETURN: 'return',
```
- [ ] **Step 4: Run — expect pass.** Also run `npm run test:run` once to confirm the union widening didn't break exhaustive switches; fix any non-exhaustive-switch type errors it surfaces (search for `satisfies NodeType` / `switch (nodeType)` over node types).
- [ ] **Step 5: Commit.**
```
git add src/types/nodes.ts src/lib/packageFormat/export-types.ts src/lib/import/type-mapping.ts src/lib/packageFormat/__tests__/node-type-mapping.test.ts
git commit -m "feat(editor): add catch/return to NodeType, StepNodeData fields, and step-type maps"
```

---

## Phase J — Unified node renderers (CORRECTED: one component per type, Tailwind + inline SVG)

### Task J1: Unified CatchNode

**Files:** Create `src/components/nodes/renderers/unified/CatchNode.tsx`. Modify `flowchart/index.ts`, `bpmn/index.ts`, `isa88/index.ts`. Test: `src/components/nodes/renderers/unified/__tests__/CatchNode.test.tsx` (create).

Grounding: existing unified nodes (e.g. `unified/StartNode.tsx`, `unified/Select1Node.tsx`) are `memo`'d default exports typed `NodeProps<Node<StepNodeData, 'x'>>`, read layout from `useNotationConfig()` (`../shared/useNotationConfig`), draw inline `<svg><polygon/></svg>`, render the label via `EditableLabel`, and place a single `<Handle>` using `config.inputPosition`/`config.outputPosition`. The CATCH shape is notation-independent (spec §2.2), so one SVG serves all three notations; only the handle position follows the notation.

- [ ] **Step 1: Failing test** — `unified/__tests__/CatchNode.test.tsx`:
```tsx
import { render } from '@testing-library/react'
import { ReactFlowProvider } from '@xyflow/react'
import { describe, it, expect } from 'vitest'
import CatchNode from '../CatchNode'

function renderNode() {
  return render(
    <ReactFlowProvider>
      <CatchNode id="c1" type="catch" selected={false} dragging={false} zIndex={0} isConnectable
        positionAbsoluteX={0} positionAbsoluteY={0}
        data={{ label: 'Recover', stepType: 'catch', catchId: 'C1' }} />
    </ReactFlowProvider>,
  )
}

describe('CatchNode', () => {
  it('renders the label and the catch_id', () => {
    const { getByText } = renderNode()
    expect(getByText('Recover')).toBeInTheDocument()
    expect(getByText('C1')).toBeInTheDocument()
  })
  it('exposes exactly one source handle (CATCH has no incoming)', () => {
    const { container } = renderNode()
    const handles = container.querySelectorAll('.react-flow__handle')
    expect(handles.length).toBe(1)
    expect(handles[0].classList.contains('react-flow__handle-source')).toBe(true)
  })
})
```
(If `EditableLabel` requires a store/provider beyond `ReactFlowProvider`, render the label as plain text in the component when `data.label` is read-only in tests — verify against how `StartNode.test.tsx`/existing node tests set up providers and mirror that exactly.)

- [ ] **Step 2: Run — expect fail** (module missing). `npm run test:run -- CatchNode`
- [ ] **Step 3: Implement** — `unified/CatchNode.tsx`:
```tsx
// Copyright (c) 2026 Saturnis.io. All rights reserved.
// Licensed under the GNU AGPL v3. See LICENSE.md for details.

import { memo } from 'react'
import { Handle, type NodeProps, type Node } from '@xyflow/react'
import { cn } from '@trajectory/ui'
import type { StepNodeData } from '@/types/nodes'
import { useNotationConfig } from '../shared/useNotationConfig'
import { EditableLabel } from '@/components/nodes/EditableLabel'
import { SELECTION_COLOR } from '../shared/colors'

type CatchNodeType = Node<StepNodeData, 'catch'>

const CATCH_FILL = '#ffe0b2'
const CATCH_STROKE = '#e65100'

/** Unified CATCH node (all 3 notations): wide-top trapezoid (a "funnel"). CATCH has only an outgoing handle. */
const CatchNode = memo(({ id, data, selected }: NodeProps<CatchNodeType>) => {
  const { outputPosition, startEnd } = useNotationConfig()
  const stroke = selected ? SELECTION_COLOR : CATCH_STROKE
  return (
    <div className="relative" style={{ width: 120, height: 50 }}>
      <svg width="120" height="50" viewBox="0 0 120 50" className="absolute inset-0">
        <polygon points="0,0 120,0 100,50 20,50" fill={CATCH_FILL} stroke={stroke} strokeWidth={selected ? 2 : 1} strokeLinejoin="round" />
      </svg>
      <div className="absolute inset-0 flex flex-col items-center justify-center pointer-events-none">
        <EditableLabel nodeId={id} label={data.label} className={cn(startEnd.labelSize, 'font-semibold text-orange-900')} />
        {data.catchId && <span className="text-[9px] font-mono text-orange-800">{data.catchId}</span>}
      </div>
      <Handle type="source" position={outputPosition} className="!w-2 !h-2 !rounded-full !bg-orange-700 !border-2 !border-white" />
    </div>
  )
})
CatchNode.displayName = 'CatchNode'
export default CatchNode
```
(Confirm `SELECTION_COLOR` is exported from `../shared/colors`; if a CATCH-specific palette constant is preferred, add `CATCH_FILL`/`CATCH_STROKE` there instead of inlining. Confirm `EditableLabel`'s prop names against `StartNode.tsx`.)

Register in each notation index (`flowchart/index.ts`, `bpmn/index.ts`, `isa88/index.ts`) — add the import and the map entry:
```typescript
import CatchNode from '../unified/CatchNode'
// ...inside flowchartNodeTypes / bpmnNodeTypes / isa88NodeTypes:
  catch: CatchNode,
```

- [ ] **Step 4: Run — expect pass.** `npm run test:run -- CatchNode`
- [ ] **Step 5: Commit.**
```
git add src/components/nodes/renderers/unified/CatchNode.tsx src/components/nodes/renderers/unified/__tests__/CatchNode.test.tsx src/components/nodes/renderers/flowchart/index.ts src/components/nodes/renderers/bpmn/index.ts src/components/nodes/renderers/isa88/index.ts
git commit -m "feat(editor): unified CatchNode (wide-top trapezoid) registered in all notations"
```

### Task J2: Unified ReturnNode

**Files:** Create `src/components/nodes/renderers/unified/ReturnNode.tsx`. Modify the three notation `index.ts` files. Test: `unified/__tests__/ReturnNode.test.tsx`.

Identical structure to J1 with these differences:
- `type ReturnNodeType = Node<StepNodeData, 'return'>`
- Narrow-top trapezoid polygon: `points="20,0 100,0 120,50 0,50"`; `RETURN_FILL = '#ffccbc'`, `RETURN_STROKE = '#bf360c'`.
- RETURN has only an **incoming** handle: `<Handle type="target" position={inputPosition} ... />` (use `const { inputPosition } = useNotationConfig()`).
- Show `data.returnConfig?.command` as the sub-label instead of `catchId`.
- Test asserts exactly one handle and that it is `react-flow__handle-target`.

- [ ] **Step 1–2:** Failing test (mirror J1; assert `getByText('ABANDON')` when `data.returnConfig.command === 'ABANDON'`, and a single target handle).
- [ ] **Step 3:** Implement `ReturnNode.tsx` (copy J1's structure with the differences above). Register `return: ReturnNode` in all three index maps.
- [ ] **Step 4:** Run — expect pass.
- [ ] **Step 5: Commit.**
```
git commit -m "feat(editor): unified ReturnNode (narrow-top trapezoid) registered in all notations"
```

> **Removed from the old plan:** no per-notation `flowchart/CatchNode.tsx`+`bpmn/CatchNode.tsx`+`isa88/CatchNode.tsx` (4–6 files), no `.module.css`, no `notation-config.ts` per-node-type dimension entries. The unified component covers all notations; geometry is in the component.

---

## Phase K — Palette & connection draw rules

### Task K1: Palette entries

**Files:** Modify `src/components/palette/StepPalette.tsx`. Test: co-located or `src/components/palette/__tests__/StepPalette.test.tsx`.

- [ ] **Step 1: Failing test** (mirror existing StepPalette tests for provider setup):
```tsx
import { render } from '@testing-library/react'
import { describe, it, expect } from 'vitest'
import { StepPalette } from '../StepPalette'

describe('StepPalette', () => {
  it('lists Catch and Return draggables', () => {
    const { getByText } = render(<StepPalette />)
    expect(getByText('Catch')).toBeInTheDocument()
    expect(getByText('Return')).toBeInTheDocument()
  })
})
```
- [ ] **Step 2: Run — expect fail.**
- [ ] **Step 3:** Append to BOTH `stepTypes` and `allStepTypes` (the exported `StepTypeItem[]` arrays):
```typescript
  { type: 'catch', label: 'Catch' },
  { type: 'return', label: 'Return' },
```
- [ ] **Step 4: Run — expect pass.**
- [ ] **Step 5: Commit.** `git commit -m "feat(editor): add Catch and Return to the step palette"`

### Task K2: Connection draw rules (CORRECTED: pure helper + extend the real callback)

**Files:** Create `src/lib/canvas/catch-network-partition.ts`, `src/lib/canvas/try-catch-connection.ts`. Modify `src/components/canvas/FlowCanvas.tsx`. Tests: `src/lib/canvas/__tests__/catch-network-partition.test.ts`, `src/lib/canvas/__tests__/try-catch-connection.test.ts`.

**K2a — port the partition helper.**
- [ ] **Step 1: Failing test** — `__tests__/catch-network-partition.test.ts`:
```typescript
import { describe, it, expect } from 'vitest'
import { partitionCatchNetworks } from '../catch-network-partition'

const steps = [
  { oid: 's1', step_type: 'START' }, { oid: 's2', step_type: 'ACTION PROXY' }, { oid: 's3', step_type: 'END' },
  { oid: 'c1', step_type: 'CATCH', catch_id: 'C1' }, { oid: 'r1', step_type: 'RETURN' },
]
const conns = [
  { from_step_id: 's1', to_step_id: 's2' }, { from_step_id: 's2', to_step_id: 's3' },
  { from_step_id: 'c1', to_step_id: 'r1' },
]

describe('partitionCatchNetworks', () => {
  it('separates the catch island (reaching a RETURN) from main flow', () => {
    const p = partitionCatchNetworks(steps, conns)
    expect([...p.catchNetworkStepOids].sort()).toEqual(['c1', 'r1'])
    expect([...p.mainFlowStepOids].sort()).toEqual(['s1', 's2', 's3'])
    expect([...(p.networksByCatchId.get('C1') ?? [])].sort()).toEqual(['c1', 'r1'])
  })
  it('does NOT count a CATCH island with no reachable RETURN', () => {
    const p = partitionCatchNetworks(
      [{ oid: 'c1', step_type: 'CATCH', catch_id: 'C1' }, { oid: 'u1', step_type: 'USER_INTERACTION' }],
      [{ from_step_id: 'c1', to_step_id: 'u1' }],
    )
    expect(p.catchNetworkStepOids.size).toBe(0)
  })
})
```
- [ ] **Step 2: Run — expect fail.**
- [ ] **Step 3: Implement** — port verbatim from the runtime (`engines/web/src/catch-network-partition.ts`); it is a pure function with no runtime deps:
```typescript
// Copyright (c) 2026 Saturnis.io. All rights reserved.
// Licensed under the GNU AGPL v3. See LICENSE.md for details.

export interface PartitionStep { oid: string; step_type: string; catch_id?: string }
export interface PartitionConnection { from_step_id: string; to_step_id: string }
export interface CatchNetworkPartition {
  mainFlowStepOids: Set<string>
  catchNetworkStepOids: Set<string>
  networksByCatchId: Map<string, Set<string>>
}

/** Spec §6.5: a catch network is the BFS-closure from a CATCH over outgoing edges, counted only if it reaches a RETURN. */
export function partitionCatchNetworks(steps: PartitionStep[], connections: PartitionConnection[]): CatchNetworkPartition {
  const outEdges = new Map<string, string[]>()
  for (const c of connections) {
    const list = outEdges.get(c.from_step_id)
    if (list) list.push(c.to_step_id)
    else outEdges.set(c.from_step_id, [c.to_step_id])
  }
  const typeByOid = new Map<string, string>()
  for (const s of steps) typeByOid.set(s.oid, s.step_type)

  const networksByCatchId = new Map<string, Set<string>>()
  const catchNetworkStepOids = new Set<string>()
  for (const step of steps) {
    if (step.step_type !== 'CATCH' || !step.catch_id) continue
    const visited = new Set<string>()
    const queue: string[] = [step.oid]
    let hasReturn = false
    while (queue.length > 0) {
      const cur = queue.shift()!
      if (visited.has(cur)) continue
      visited.add(cur)
      if (typeByOid.get(cur) === 'RETURN') hasReturn = true
      for (const n of outEdges.get(cur) ?? []) if (!visited.has(n)) queue.push(n)
    }
    if (hasReturn) {
      networksByCatchId.set(step.catch_id, visited)
      for (const oid of visited) catchNetworkStepOids.add(oid)
    }
  }
  const mainFlowStepOids = new Set<string>()
  for (const s of steps) if (!catchNetworkStepOids.has(s.oid)) mainFlowStepOids.add(s.oid)
  return { mainFlowStepOids, catchNetworkStepOids, networksByCatchId }
}
```
- [ ] **Step 4: Run — expect pass. Step 5: Commit.** `git commit -m "feat(editor): port catch-network partition helper"`

**K2b — pure connection-rule helper.**
- [ ] **Step 1: Failing test** — `__tests__/try-catch-connection.test.ts`:
```typescript
import { describe, it, expect } from 'vitest'
import { isTryCatchConnectionValid } from '../try-catch-connection'
import type { WorkflowNode } from '@/types/nodes'

const nodes = [
  { id: 's1', type: 'start', data: { label: 'S', stepType: 'start' }, position: { x: 0, y: 0 } },
  { id: 's2', type: 'end', data: { label: 'E', stepType: 'end' }, position: { x: 0, y: 0 } },
  { id: 'c1', type: 'catch', data: { label: 'C', stepType: 'catch', catchId: 'C1' }, position: { x: 0, y: 0 } },
  { id: 'r1', type: 'return', data: { label: 'R', stepType: 'return', returnConfig: { command: 'ABANDON' } }, position: { x: 0, y: 0 } },
] as unknown as WorkflowNode[]
const edges = [{ id: 'e1', source: 's1', target: 's2' }, { id: 'e2', source: 'c1', target: 'r1' }]

describe('isTryCatchConnectionValid', () => {
  it('rejects an incoming edge to CATCH', () => expect(isTryCatchConnectionValid({ source: 's1', target: 'c1' }, nodes, edges)).toBe(false))
  it('rejects an outgoing edge from RETURN', () => expect(isTryCatchConnectionValid({ source: 'r1', target: 's2' }, nodes, edges)).toBe(false))
  it('rejects a cross-network edge (main → catch island)', () => expect(isTryCatchConnectionValid({ source: 's2', target: 'r1' }, nodes, edges)).toBe(false))
  it('allows a main-flow edge', () => expect(isTryCatchConnectionValid({ source: 's1', target: 's2' }, nodes, edges)).toBe(true))
  it('allows an edge within a catch network', () => expect(isTryCatchConnectionValid({ source: 'c1', target: 'r1' }, nodes, edges)).toBe(true))
})
```
- [ ] **Step 2: Run — expect fail.**
- [ ] **Step 3: Implement** — `src/lib/canvas/try-catch-connection.ts`:
```typescript
// Copyright (c) 2026 Saturnis.io. All rights reserved.
// Licensed under the GNU AGPL v3. See LICENSE.md for details.

import type { Edge } from '@xyflow/react'
import type { WorkflowNode } from '@/types/nodes'
import { NODE_TYPE_TO_STEP_TYPE } from '@/lib/packageFormat/export-types'
import { partitionCatchNetworks, type PartitionStep } from './catch-network-partition'

/** Returns false if the proposed edge violates a TRY/CATCH/RETURN structural rule. */
export function isTryCatchConnectionValid(
  connection: { source: string; target: string },
  nodes: WorkflowNode[],
  edges: Pick<Edge, 'source' | 'target'>[],
): boolean {
  const source = nodes.find((n) => n.id === connection.source)
  const target = nodes.find((n) => n.id === connection.target)
  if (!source || !target) return false

  // Structural: CATCH takes no incoming; RETURN emits no outgoing.
  if (target.type === 'catch') return false
  if (source.type === 'return') return false

  // Cross-network: an edge may not straddle the main flow / a catch island.
  const steps: PartitionStep[] = nodes.map((n) => ({
    oid: n.id,
    step_type: NODE_TYPE_TO_STEP_TYPE[(n.type ?? 'start') as keyof typeof NODE_TYPE_TO_STEP_TYPE] ?? 'UNKNOWN',
    catch_id: (n.data as { catchId?: string }).catchId,
  }))
  const conns = edges.map((e) => ({ from_step_id: e.source, to_step_id: e.target }))
  const part = partitionCatchNetworks(steps, conns)
  const sInCatch = part.catchNetworkStepOids.has(connection.source)
  const tInCatch = part.catchNetworkStepOids.has(connection.target)
  if (sInCatch !== tInCatch) return false

  return true
}
```
- [ ] **Step 4: Run — expect pass. Step 5: Commit.** `git commit -m "feat(editor): pure TRY/CATCH connection-rule helper"`

**K2c — wire into the real callback.** In `src/components/canvas/FlowCanvas.tsx`, the existing `isValidConnection` is a `useCallback` reading `nodes`/`edges` from `useFlowStore`. Add the TRY/CATCH check before the final `return true`:
- [ ] **Step 1:** Import the helper: `import { isTryCatchConnectionValid } from '@/lib/canvas/try-catch-connection'`.
- [ ] **Step 2:** Inside the callback, after the existing duplicate-edge check and before `return true`, add:
```typescript
    // TRY/CATCH/RETURN structural + cross-network rules
    if (!isTryCatchConnectionValid({ source: connection.source, target: connection.target }, nodes, edges)) {
      return false
    }
```
  (Keep the callback's single `connection` arg and its `[nodes, edges]` dependency array. `connection.source`/`target` are `string | null` in React Flow's `Connection`; the existing code already finds nodes by them, so guard as it does — if your TS complains, coerce with `connection.source ?? ''`.)
- [ ] **Step 3: Manual check** — `npm run dev`, draw an edge into a CATCH and out of a RETURN; both should be refused; a CATCH→RETURN edge should be allowed.
- [ ] **Step 4: Commit.** `git commit -m "feat(editor): block CATCH-incoming, RETURN-outgoing, cross-network edges at draw time"`

---

## Phase L — Property-panel editors

> Grounding: `NodeInlineEditor.tsx` exposes `export const PARAMETERIZED_TYPES: NodeType[]` and `const CONTROL_FLOW_TYPES: NodeType[]`, edits via a local `draft` + `updateDraft(patch)`, and renders `ResourceCommandEditor` for every `!CONTROL_FLOW_TYPES.includes(nodeType)`. Add `'catch'` to `PARAMETERIZED_TYPES` (so its output-parameter/trigger editor shows) and `'return'` to `CONTROL_FLOW_TYPES` (so RETURN shows no resource/param editors). Derive available catches from `useFlowStore`.

### Task L1: TryEditor (on ACTION PROXY steps)

**Files:** Create `src/components/properties/TryEditor.tsx`, `__tests__/TryEditor.test.tsx`. Modify `NodeInlineEditor.tsx`.

- [ ] **Step 1: Failing tests** — `__tests__/TryEditor.test.tsx`:
```tsx
import { render, fireEvent } from '@testing-library/react'
import { describe, it, expect, vi } from 'vitest'
import { TryEditor } from '../TryEditor'

const catches = [{ catchId: 'OvenFailed' }, { catchId: 'OperatorAborted' }]

describe('TryEditor', () => {
  it('shows an add button and no rows when empty', () => {
    const { getByText, queryByRole } = render(<TryEditor value={[]} onChange={vi.fn()} availableCatches={catches} />)
    expect(getByText(/Add TRY/i)).toBeInTheDocument()
    expect(queryByRole('listitem')).toBeNull()
  })
  it('add appends a row with the first unused mode + first catch', () => {
    const onChange = vi.fn()
    const { getByText } = render(<TryEditor value={[]} onChange={onChange} availableCatches={catches} />)
    fireEvent.click(getByText(/Add TRY/i))
    expect(onChange).toHaveBeenCalledWith([{ mode: 'ERROR', catch_id: 'OvenFailed', release_on_catch: true }])
  })
  it('toggling release_on_catch updates the row', () => {
    const onChange = vi.fn()
    const { getByRole } = render(
      <TryEditor value={[{ mode: 'ERROR', catch_id: 'OvenFailed', release_on_catch: true }]} onChange={onChange} availableCatches={catches} />,
    )
    fireEvent.click(getByRole('checkbox'))
    expect(onChange).toHaveBeenCalledWith([{ mode: 'ERROR', catch_id: 'OvenFailed', release_on_catch: false }])
  })
})
```
- [ ] **Step 2: Run — expect fail.**
- [ ] **Step 3: Implement** — `TryEditor.tsx` (Tailwind, no CSS modules; props mirror `ResourceCommandEditor`'s `{value/onChange}` shape):
```tsx
// Copyright (c) 2026 Saturnis.io. All rights reserved.
// Licensed under the GNU AGPL v3. See LICENSE.md for details.

const ALL_MODES = ['ERROR', 'ABORT', 'TIMEOUT'] as const
export interface TrySpec { mode: 'ERROR' | 'ABORT' | 'TIMEOUT'; catch_id: string; release_on_catch?: boolean }
interface Props { value: TrySpec[]; onChange: (next: TrySpec[]) => void; availableCatches: Array<{ catchId: string }> }

export function TryEditor({ value, onChange, availableCatches }: Props) {
  const used = new Set(value.map((t) => t.mode))
  const firstMode = ALL_MODES.find((m) => !used.has(m)) ?? 'ERROR'
  const firstCatch = availableCatches[0]?.catchId ?? ''
  const add = () => onChange([...value, { mode: firstMode, catch_id: firstCatch, release_on_catch: true }])
  const update = (i: number, patch: Partial<TrySpec>) => onChange(value.map((t, j) => (j === i ? { ...t, ...patch } : t)))
  const remove = (i: number) => onChange(value.filter((_, j) => j !== i))

  return (
    <section className="mt-3 border-t pt-3">
      <div className="flex items-center justify-between">
        <h4 className="text-xs font-semibold text-muted-foreground">TRY · error handling</h4>
        <button type="button" onClick={add} disabled={used.size === ALL_MODES.length}
          className="text-xs rounded border px-2 py-0.5 disabled:opacity-50">+ Add TRY</button>
      </div>
      <ul role="list" className="mt-2 space-y-2">
        {value.map((row, i) => (
          <li role="listitem" key={i} className="flex flex-wrap items-center gap-2 text-xs">
            <label className="flex items-center gap-1">ON
              <select value={row.mode} onChange={(e) => update(i, { mode: e.target.value as TrySpec['mode'] })}>
                {ALL_MODES.filter((m) => m === row.mode || !used.has(m)).map((m) => <option key={m} value={m}>{m}</option>)}
              </select>
            </label>
            <label className="flex items-center gap-1">CATCH
              <select value={row.catch_id} onChange={(e) => update(i, { catch_id: e.target.value })}>
                {availableCatches.map((c) => <option key={c.catchId} value={c.catchId}>{c.catchId}</option>)}
              </select>
            </label>
            <label className="flex items-center gap-1">
              <input type="checkbox" checked={row.release_on_catch !== false}
                onChange={(e) => update(i, { release_on_catch: e.target.checked })} />
              Release resources on catch
            </label>
            <button type="button" aria-label="Remove TRY" onClick={() => remove(i)}>✕</button>
          </li>
        ))}
      </ul>
    </section>
  )
}
```
- [ ] **Step 4: Wire into `NodeInlineEditor.tsx`.** Near the top, derive catches from the store:
```tsx
import { useFlowStore } from '@/stores/flowStore'
import { TryEditor } from './TryEditor'
// inside the component:
const availableCatches = useFlowStore((s) =>
  s.nodes.filter((n) => n.type === 'catch' && (n.data as { catchId?: string }).catchId)
    .map((n) => ({ catchId: (n.data as { catchId: string }).catchId })))
```
  Then, in the `actionProxyWait` area (next to where `ResourceCommandEditor` is rendered for non-control-flow types), add:
```tsx
{nodeType === 'actionProxyWait' && (
  <TryEditor
    value={(draft.trySpecifications ?? []) as TrySpec[]}
    onChange={(ts) => updateDraft({ trySpecifications: ts })}
    availableCatches={availableCatches}
  />
)}
```
  (Match the real local variable names — `draft`/`updateDraft` — confirmed in `NodeInlineEditor.tsx`. If the action editor lives in a sub-component rather than inline, place `TryEditor` there, after the resource-command block.)
- [ ] **Step 5: Run — expect pass. Step 6: Commit.** `git commit -m "feat(editor): TryEditor section on action steps"`

### Task L2: ReturnConfigEditor

**Files:** Create `src/components/properties/ReturnConfigEditor.tsx`, `__tests__/ReturnConfigEditor.test.tsx`. Modify `NodeInlineEditor.tsx`.

Behavior: one command `<select>` (ABANDON/RESTART/GOTO/RETRY). Conditional fields:
- RESTART → `restart_mode` select (CLEAN/KEEP), required (show empty-invalid if unset).
- GOTO → `goto_step_oid` select populated from **main-flow** nodes only — i.e. nodes where `partitionCatchNetworks(...).mainFlowStepOids.has(node.id)` AND type ∉ {`catch`,`return`,`start`}. Pass the candidate list in as a prop (compute it in `NodeInlineEditor` from `useFlowStore` + the ported partition helper).
- ABANDON / RETRY → no extra fields.

- [ ] **Step 1: Failing tests** — render each command; switching command resets stale sub-fields; GOTO lists the provided main-flow targets; assert `onChange` payloads (`{command:'RESTART', restart_mode:'KEEP'}`, `{command:'GOTO', goto_step_oid:'m1'}`).
- [ ] **Step 2–4:** Implement `ReturnConfigEditor` (`{ value: ReturnConfig; onChange; gotoTargets: Array<{id:string;label:string}> }`), wire into the `nodeType === 'return'` branch of `NodeInlineEditor`, passing `gotoTargets` derived from the store + partition.
- [ ] **Step 5: Commit.** `git commit -m "feat(editor): ReturnConfigEditor for RETURN steps"`

### Task L3: CatchIdEditor + node-type registration

**Files:** Create `src/components/properties/CatchIdEditor.tsx`, `__tests__/CatchIdEditor.test.tsx`. Modify `NodeInlineEditor.tsx`.

- [ ] **Step 1:** Add `'catch'` to `PARAMETERIZED_TYPES` and `'return'` to `CONTROL_FLOW_TYPES` in `NodeInlineEditor.tsx`.
- [ ] **Step 2: Failing test** — a single text input bound to `catchId`; typing an id already used by another CATCH shows a duplicate warning (red border + message). Use `availableCatches` (minus the current node) for the uniqueness check.
- [ ] **Step 3:** Implement `CatchIdEditor` (`{ value: string; onChange: (id: string) => void; takenIds: string[] }`). Wire into the `nodeType === 'catch'` branch; its `onChange` calls the store's `renameCatchId` (Phase O) so references update atomically rather than a bare `updateDraft({ catchId })`.
- [ ] **Step 4: Run — expect pass. Step 5: Commit.** `git commit -m "feat(editor): CatchIdEditor with live uniqueness check"`

---

## Phase M — Exporter & importer

> **Prerequisite (Step 0):** confirm the editor's bundled workflow schema (`src/lib/import/schema-validator.ts` / the AJV schema it loads) accepts `try_specifications`, `catch_id`, `return_config`, and the `CATCH`/`RETURN` step types. If `validateWorkflowSpecification` rejects a workflow carrying them, add them to the schema first and commit `chore(editor): allow CATCH/RETURN + try_specifications/catch_id/return_config in import schema`. The round-trip test in M2 will fail at import if this is missed.

### Task M1: step-extractors

**Files:** Modify `src/lib/packageFormat/step-extractors.ts`, `__tests__/step-extractors.test.ts` (create if absent).

- [ ] **Step 1: Failing tests** (mirror the existing standalone-extractor tests):
```typescript
import { describe, it, expect } from 'vitest'
import { extractTrySpecifications, extractReturnConfig, extractCatchId } from '../step-extractors'

describe('extractTrySpecifications', () => {
  it('undefined when none', () => expect(extractTrySpecifications({})).toBeUndefined())
  it('passes through valid rows', () =>
    expect(extractTrySpecifications({ trySpecifications: [{ mode: 'ERROR', catch_id: 'C', release_on_catch: true }] }))
      .toEqual([{ mode: 'ERROR', catch_id: 'C', release_on_catch: true }]))
  it('drops invalid modes', () =>
    expect(extractTrySpecifications({ trySpecifications: [{ mode: 'WAT', catch_id: 'C' }] })).toBeUndefined())
})
describe('extractReturnConfig', () => {
  it('ABANDON', () => expect(extractReturnConfig({ returnConfig: { command: 'ABANDON' } })).toEqual({ command: 'ABANDON' }))
  it('RESTART+mode', () => expect(extractReturnConfig({ returnConfig: { command: 'RESTART', restart_mode: 'CLEAN' } }))
    .toEqual({ command: 'RESTART', restart_mode: 'CLEAN' }))
})
describe('extractCatchId', () => {
  it('returns the string', () => expect(extractCatchId({ catchId: 'OvenFailed' })).toBe('OvenFailed'))
})
```
- [ ] **Step 2: Run — expect fail.**
- [ ] **Step 3: Implement** (append, following the file's `export function extractX(data: Record<string, unknown>)` convention):
```typescript
const VALID_TRY_MODES = new Set(['ERROR', 'ABORT', 'TIMEOUT'])
export function extractTrySpecifications(data: Record<string, unknown>) {
  const ts = data?.trySpecifications
  if (!Array.isArray(ts) || ts.length === 0) return undefined
  const cleaned = ts.filter((t) => t && VALID_TRY_MODES.has((t as { mode?: string }).mode ?? '') && typeof (t as { catch_id?: unknown }).catch_id === 'string')
  return cleaned.length ? cleaned : undefined
}
export function extractReturnConfig(data: Record<string, unknown>) {
  const rc = data?.returnConfig as { command?: string } | undefined
  return rc?.command ? rc : undefined
}
export function extractCatchId(data: Record<string, unknown>): string | undefined {
  return typeof data?.catchId === 'string' ? (data.catchId as string) : undefined
}
```
- [ ] **Step 4: Run — expect pass. Step 5: Commit.** `git commit -m "feat(editor): extractTrySpecifications/extractReturnConfig/extractCatchId"`

### Task M2: exporter + importer wiring + round-trip (CORRECTED signatures)

**Files:** Modify `src/lib/packageFormat/workflow-exporters.ts`, `src/lib/import/workflow-importers.ts`. Test: `src/lib/packageFormat/__tests__/try-catch-roundtrip.test.ts`.

- [ ] **Step 1: Failing round-trip test** (uses the REAL `exportWorkflow(spec, nodes, edges)` and `importWorkflow(json): Success|Error`):
```typescript
import { describe, it, expect } from 'vitest'
import { exportWorkflow } from '../workflow-exporters'
import { importWorkflow } from '../../import/workflow-importers'
import type { WorkflowNode } from '@/types/nodes'

const spec = { id: 'wf-rt', libraryId: 'lib', name: 'RT', description: '', version: '1.0.0', state: 'Draft' as const, data: {}, createdAt: 0, updatedAt: 0 }
const nodes = [
  { id: 's1', type: 'start', position: { x: 0, y: 0 }, data: { label: 'Start', stepType: 'start' } },
  { id: 's2', type: 'actionProxyWait', position: { x: 0, y: 80 }, data: { label: 'Action', stepType: 'actionProxyWait',
    actionProxyConfig: { action_oid: 'act-1', environment_oid: 'env-1' },
    trySpecifications: [{ mode: 'ERROR', catch_id: 'C1', release_on_catch: true }] } },
  { id: 's3', type: 'end', position: { x: 0, y: 160 }, data: { label: 'End', stepType: 'end' } },
  { id: 'c1', type: 'catch', position: { x: 200, y: 0 }, data: { label: 'Recover', stepType: 'catch', catchId: 'C1' } },
  { id: 'r1', type: 'return', position: { x: 200, y: 80 }, data: { label: 'Done', stepType: 'return', returnConfig: { command: 'ABANDON' } } },
] as unknown as WorkflowNode[]
const edges = [
  { id: 'e1', source: 's1', target: 's2' }, { id: 'e2', source: 's2', target: 's3' }, { id: 'e3', source: 'c1', target: 'r1' },
]

describe('TRY/CATCH/RETURN export → import round-trip', () => {
  it('preserves trySpecifications, catchId, returnConfig', () => {
    const exported = exportWorkflow(spec as never, nodes, edges as never)
    const result = importWorkflow(exported)
    if ('error' in result) throw new Error(result.error)
    const find = (id: string) => result.nodes.find((n) => n.id === id)!
    expect((find('s2').data as { trySpecifications?: unknown }).trySpecifications)
      .toEqual([{ mode: 'ERROR', catch_id: 'C1', release_on_catch: true }])
    expect((find('c1').data as { catchId?: string }).catchId).toBe('C1')
    expect((find('r1').data as { returnConfig?: unknown }).returnConfig).toEqual({ command: 'ABANDON' })
  })
})
```
  (Node ids are regenerated on import in some paths; this test re-imports the *exported wire JSON* whose step `oid`s are stable, so `find` by id works. If `importWorkflow` reassigns ids, compare by `step_type`/order instead — verify against `transformWorkflowSpec`.)
- [ ] **Step 2: Run — expect fail.**
- [ ] **Step 3: Exporter** — in `workflow-exporters.ts`, inside the `nodes.map((node) => ({ ... }))` step object (where `action_proxy_config` is already emitted), add:
```typescript
        try_specifications: extractTrySpecifications(data),
        catch_id: nodeType === 'catch' ? extractCatchId(data) : undefined,
        return_config: nodeType === 'return' ? extractReturnConfig(data) : undefined,
```
  (Import the three extractors at the top alongside the existing ones. `data` = `node.data as StepNodeData`, `nodeType = data.stepType` — already in scope.)
- [ ] **Step 4: Importer** — in `transformWorkflowSpec`'s per-step `nodeData` object (`workflow-importers.ts`), add:
```typescript
      ...(step.try_specifications ? { trySpecifications: step.try_specifications } : {}),
      ...(step.catch_id ? { catchId: step.catch_id } : {}),
      ...(step.return_config ? { returnConfig: step.return_config } : {}),
```
  (Mirror how `actionProxyConfig` is conditionally spread. Add `try_specifications`/`catch_id`/`return_config` to the `MasterWorkflowStep` type in `export-types.ts` if TS reports them missing.)
- [ ] **Step 5: Run — expect pass. Step 6: Commit.** `git commit -m "feat(editor): export/import round-trip for trySpecifications/catchId/returnConfig"`

---

## Phase N — Validation (CORRECTED: message-based, mirrored locally)

> The editor validator (`src/lib/export/workflow-validation.ts`) returns `ValidationError[]` (`{nodeId,nodeName,nodeType,message}`), surfaced by `validateWorkflowWithChildren` → `ValidationErrorDialog` (used in `CanvasToolbar.tsx`). There is **no importable runtime validator**, so we mirror the rules as editor messages. The editor is also the right place for the editor-time checks the runtime omits (a CATCH that never reaches a RETURN, an unmatched TRY, a GOTO into a catch island).

### Task N1: degree rules + TRY/CATCH/RETURN structural messages

**Files:** Modify `src/lib/export/workflow-validation.ts`. Test: `src/lib/export/__tests__/try-catch-validation.test.ts`.

- [ ] **Step 1: Failing tests** — assert a `ValidationError` (by substring of `message`) for each violation, and zero for a well-formed workflow:
```typescript
import { describe, it, expect } from 'vitest'
import { validateWorkflow } from '../workflow-validation'
import type { WorkflowNode } from '@/types/nodes'
import type { Edge } from '@xyflow/react'

const N = (id: string, type: string, data: Record<string, unknown> = {}): WorkflowNode =>
  ({ id, type, position: { x: 0, y: 0 }, data: { label: id, stepType: type, ...data } } as unknown as WorkflowNode)
const E = (source: string, target: string): Edge => ({ id: `${source}-${target}`, source, target } as Edge)
const msgs = (n: WorkflowNode[], e: Edge[]) => validateWorkflow(n, e).map((x) => x.message).join(' | ')

describe('TRY/CATCH/RETURN validation', () => {
  it('flags a duplicate catch_id', () => {
    const nodes = [N('s1','start'), N('s2','end'), N('c1','catch',{catchId:'C1'}), N('r1','return',{returnConfig:{command:'ABANDON'}}),
                   N('c2','catch',{catchId:'C1'}), N('r2','return',{returnConfig:{command:'ABANDON'}})]
    const edges = [E('s1','s2'), E('c1','r1'), E('c2','r2')]
    expect(msgs(nodes, edges)).toMatch(/duplicate catch_id|catch_id .*C1/i)
  })
  it('flags an unmatched TRY (catch_id with no CATCH)', () => {
    const nodes = [N('s1','start'), N('s2','actionProxyWait',{trySpecifications:[{mode:'ERROR',catch_id:'Ghost'}]}), N('s3','end')]
    expect(msgs(nodes, [E('s1','s2'),E('s2','s3')])).toMatch(/unmatched try|no CATCH|Ghost/i)
  })
  it('flags a CATCH with no reachable RETURN', () => {
    const nodes = [N('s1','start'), N('s2','end'), N('c1','catch',{catchId:'C1'}), N('u1','userInteractionStep')]
    expect(msgs(nodes, [E('s1','s2'), E('c1','u1')])).toMatch(/RETURN/i)
  })
  it('flags a GOTO target inside a catch network', () => {
    const nodes = [N('s1','start'), N('s2','actionProxyWait',{trySpecifications:[{mode:'ERROR',catch_id:'C1'}]}), N('s3','end'),
                   N('c1','catch',{catchId:'C1'}), N('r1','return',{returnConfig:{command:'GOTO',goto_step_oid:'c1'}})]
    expect(msgs(nodes, [E('s1','s2'),E('s2','s3'),E('c1','r1')])).toMatch(/GOTO.*catch|goto_step_oid/i)
  })
  it('passes a well-formed try/catch/return workflow', () => {
    const nodes = [N('s1','start'), N('s2','actionProxyWait',{trySpecifications:[{mode:'ERROR',catch_id:'C1'}]}), N('s3','end'),
                   N('c1','catch',{catchId:'C1'}), N('r1','return',{returnConfig:{command:'ABANDON'}})]
    const tc = validateWorkflow(nodes, [E('s1','s2'),E('s2','s3'),E('c1','r1')])
      .filter((x) => x.nodeType === 'catch' || x.nodeType === 'return' || /TRY|CATCH|RETURN/i.test(x.message))
    expect(tc).toEqual([])
  })
})
```
- [ ] **Step 2: Run — expect fail.**
- [ ] **Step 3: Implement.**
  - Add `catch`/`return` to `NODE_RULES` (mirroring START/END degree, so the generic loop validates connectivity):
```typescript
  catch: { minIncoming: 0, maxIncoming: 0, minOutgoing: 1, maxOutgoing: 1 },
  return: { minIncoming: 1, maxIncoming: Infinity, minOutgoing: 0, maxOutgoing: 0 },
```
  - Exempt `catch` from the disconnected-node check (it legitimately has 0 incoming, like `start`): change `nodeType !== 'start' && nodeType !== 'end'` to also exclude `'catch'`.
  - Add a `validateTryCatchReturn(nodes, edges): ValidationError[]` function (uses the ported partition helper) and push its results inside `validateWorkflow` before `return errors`:
```typescript
import { partitionCatchNetworks, type PartitionStep } from '@/lib/canvas/catch-network-partition'
import { NODE_TYPE_TO_STEP_TYPE } from '@/lib/packageFormat/export-types'

function validateTryCatchReturn(nodes: WorkflowNode[], edges: Edge[]): ValidationError[] {
  const errors: ValidationError[] = []
  const err = (n: WorkflowNode, message: string): ValidationError =>
    ({ nodeId: n.id, nodeName: (n.data as { label?: string }).label || n.id, nodeType: n.type || 'unknown', message })

  const catchNodes = nodes.filter((n) => n.type === 'catch')
  const catchIds = new Map<string, WorkflowNode[]>()
  for (const c of catchNodes) {
    const id = (c.data as { catchId?: string }).catchId
    if (!id) { errors.push(err(c, 'CATCH has no catch_id')); continue }
    const list = catchIds.get(id) ?? []; list.push(c); catchIds.set(id, list)
  }
  // DUPLICATE_CATCH_ID
  for (const [id, list] of catchIds) if (list.length > 1) for (const c of list) errors.push(err(c, `Duplicate catch_id "${id}" — each CATCH must be unique`))

  // Partition for cross-network + reachability + GOTO checks
  const steps: PartitionStep[] = nodes.map((n) => ({ oid: n.id,
    step_type: NODE_TYPE_TO_STEP_TYPE[(n.type ?? 'start') as keyof typeof NODE_TYPE_TO_STEP_TYPE] ?? 'UNKNOWN',
    catch_id: (n.data as { catchId?: string }).catchId }))
  const conns = edges.map((e) => ({ from_step_id: e.source, to_step_id: e.target }))
  const part = partitionCatchNetworks(steps, conns)

  // CATCH_WITHOUT_RETURN: a CATCH whose island never reaches a RETURN won't appear in networksByCatchId
  for (const c of catchNodes) {
    const id = (c.data as { catchId?: string }).catchId
    if (id && !part.networksByCatchId.has(id)) errors.push(err(c, 'CATCH network has no reachable RETURN'))
  }

  // UNMATCHED_TRY: a TRY referencing a catch_id with no CATCH
  const knownCatchIds = new Set([...catchIds.keys()])
  for (const n of nodes) {
    const ts = (n.data as { trySpecifications?: Array<{ mode: string; catch_id: string }> }).trySpecifications
    if (!Array.isArray(ts)) continue
    for (const t of ts) if (!knownCatchIds.has(t.catch_id)) errors.push(err(n, `TRY ON ${t.mode} references unknown catch_id "${t.catch_id}"`))
    if (n.type !== 'actionProxyWait' && ts.length) errors.push(err(n, 'TRY is only allowed on ACTION PROXY steps')) // TRY_ON_INVALID_STEP
  }

  // GOTO_TARGET_IN_CATCH
  for (const n of nodes.filter((x) => x.type === 'return')) {
    const rc = (n.data as { returnConfig?: { command?: string; goto_step_oid?: string } }).returnConfig
    if (rc?.command === 'GOTO' && rc.goto_step_oid && part.catchNetworkStepOids.has(rc.goto_step_oid))
      errors.push(err(n, `GOTO target "${rc.goto_step_oid}" is inside a catch network`))
  }
  return errors
}
```
  Call it: `errors.push(...validateTryCatchReturn(nodes, edges))` just before `return errors` in `validateWorkflow`.
- [ ] **Step 4: Run — expect pass.** Also `npm run test:run` to confirm no existing validation test regressed.
- [ ] **Step 5: Manual check** — `npm run dev`, build a malformed try/catch and confirm the messages appear in the `ValidationErrorDialog` (via the toolbar validate action).
- [ ] **Step 6: Commit.** `git commit -m "feat(editor): message-based TRY/CATCH/RETURN validation rules"`

---

## Phase O — Change-propagation (CORRECTED: flowStore action)

### Task O1: `renameCatchId` in flowStore

**Files:** Modify `src/stores/flowStore.ts`, `src/stores/types.ts`. Test: `src/stores/__tests__/catch-rename-propagation.test.ts`.

- [ ] **Step 1: Failing test:**
```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { useFlowStore } from '../flowStore'
import type { WorkflowNode } from '@/types/nodes'

describe('renameCatchId propagation', () => {
  beforeEach(() => {
    useFlowStore.setState({
      nodes: [
        { id: 'a1', type: 'actionProxyWait', position: { x: 0, y: 0 },
          data: { label: 'A', stepType: 'actionProxyWait', trySpecifications: [{ mode: 'ERROR', catch_id: 'OldId', release_on_catch: true }] } },
        { id: 'c1', type: 'catch', position: { x: 0, y: 0 }, data: { label: 'C', stepType: 'catch', catchId: 'OldId' } },
      ] as unknown as WorkflowNode[],
      edges: [],
      hydratedSpecId: null, // suppress write-through in tests (writeThroughToSpec early-returns)
    })
  })
  it('renames the CATCH and every referencing TRY', () => {
    useFlowStore.getState().renameCatchId('OldId', 'NewId')
    const nodes = useFlowStore.getState().nodes
    expect((nodes.find((n) => n.id === 'c1')!.data as { catchId: string }).catchId).toBe('NewId')
    expect((nodes.find((n) => n.id === 'a1')!.data as { trySpecifications: Array<{ catch_id: string }> }).trySpecifications[0].catch_id).toBe('NewId')
  })
})
```
- [ ] **Step 2: Run — expect fail.**
- [ ] **Step 3: Implement** — add to the `FlowState` interface in `src/stores/types.ts`:
```typescript
  renameCatchId: (oldId: string, newId: string) => void
```
  and the action in `flowStore.ts` (alongside `updateNodeData`, ending with `writeThroughToSpec()` so it persists + lands as one zundo snapshot):
```typescript
        renameCatchId: (oldId, newId) => {
          if (oldId === newId) return
          set({
            nodes: get().nodes.map((n) => {
              if (n.type === 'catch' && (n.data as { catchId?: string }).catchId === oldId) {
                return { ...n, data: { ...n.data, catchId: newId } }
              }
              const ts = (n.data as { trySpecifications?: Array<{ catch_id: string }> }).trySpecifications
              if (Array.isArray(ts) && ts.some((t) => t.catch_id === oldId)) {
                return { ...n, data: { ...n.data, trySpecifications: ts.map((t) => (t.catch_id === oldId ? { ...t, catch_id: newId } : t)) } }
              }
              return n
            }),
          })
          writeThroughToSpec()
        },
```
  Wire `CatchIdEditor.onChange` (L3) to call `useFlowStore.getState().renameCatchId(prevId, nextId)` so a single edit updates the CATCH + all references in one undo step (the 500ms zundo debounce groups it).
- [ ] **Step 4: Run — expect pass. Step 5: Commit.** `git commit -m "feat(editor): renameCatchId propagates to all TRY references in one undo"`

### Task O2: deleting a CATCH surfaces an unmatched-TRY error

No special store action is needed — deleting a CATCH (normal `deleteSelectedNodes`) leaves its referencing TRYs dangling, which Phase N's `UNMATCHED_TRY` check now flags.
- [ ] **Step 1: Test** — in a Vitest test, set flowStore nodes to a TRY referencing `C1` plus a CATCH `C1`, delete the CATCH (`useFlowStore.setState` removing it), run `validateWorkflow(nodes, edges)` and assert an `/unknown catch_id "C1"/i` message appears.
- [ ] **Step 2: Run — expect pass** (the rule already exists from N1; this test locks the behavior).
- [ ] **Step 3: Commit.** `git commit -m "test(editor): deleting a referenced CATCH surfaces UNMATCHED_TRY"`

---

## Phase P — Integration

### Task P1: end-to-end smoke + suite

- [ ] **Step 1:** `npm run dev`. Draw: START → ACTION PROXY (TRY ON ERROR → C1) → END; plus CATCH C1 → USER_INTERACTION → RETURN ABANDON. Confirm the connection rules block illegal edges and the validation dialog is clean.
- [ ] **Step 2:** Export to `.WFmasterX`; open the `.WFmaster` JSON and confirm `try_specifications` on the action, `step_type:"CATCH"` + `catch_id`, `step_type:"RETURN"` + `return_config:{command:"ABANDON"}`.
- [ ] **Step 3:** Re-import the `.WFmasterX`; confirm the canvas reproduces the workflow (step types, edges, TRY/catch_id/return_config intact).
- [ ] **Step 4:** Full gates: `npm run test:run && npm run build` (the build runs `tsc -b`, catching the NodeType-union exhaustiveness and any type drift) `&& npm run lint`. Fix failures.
- [ ] **Step 5: Commit** any cleanup. `git commit -m "chore(editor): typecheck/lint clean after TRY/CATCH/RETURN"`
- [ ] **Step 6:** Push (per session conventions, confirm with the user before pushing if on `main`).

---

## Self-Review

**1. Spec coverage**
| Spec section | Task(s) |
|---|---|
| §2.1 palette | K1 |
| §2.2 node renderers (3 notations) | J1–J2 (unified) |
| §2.3 connection draw rules | K2 |
| §2.4 PropertyPanel additions | L1–L3 |
| §2.5 change-propagation | O1, O2 |
| §2.6 envelope mappings | I1, M1, M2 |
| §6 validator rules (editor-time, message form) | N1 |

**2. The 6 corrections, each landed in a task**
1. Store → flowStore: O1 (+ test uses `useFlowStore`). 2. `isValidConnection` real signature: K2c (extends the FlowCanvas callback; pure helper in K2b). 3. Validator message-based + local mirror + local partition: N1, K2a. 4. Unified renderers + Tailwind/SVG: J1–J2. 5. No `notation-config` per-node-type edit: geometry in J1/J2 components. 6. Real exporter/importer signatures: M2 round-trip test.

**3. Placeholder scan:** code shown for every implementation step; soft references ("confirm against StartNode.tsx", "match `draft`/`updateDraft`") name the exact real file/pattern to mirror, because component-internal conventions (provider setup in tests, the draft mechanism) must be matched to the live code rather than guessed.

**4. Type consistency:** editor side `trySpecifications`/`catchId`/`returnConfig` (camelCase, on `StepNodeData`) ↔ wire `try_specifications`/`catch_id`/`return_config` (snake_case); `TrySpec.catch_id` is snake_case on BOTH sides (it is an inner field of the array, mirroring how `actionProxyConfig.action_oid` stays snake_case inside the editor object). Extractors/importers translate only the outer field name. `NODE_TYPE_TO_STEP_TYPE` (forward) and `STEP_TYPE_TO_NODE_TYPE` (reverse) both updated.

**Open implementer decisions (genuinely environment-dependent, not placeholders):**
- Whether CATCH colours go in `../shared/colors` or inline in the component (J1) — cosmetic.
- Whether to memoize the partition inside `validateTryCatchReturn` / `isTryCatchConnectionValid` if canvas perf shows it (both recompute per call; fine at typical workflow sizes).
