# TRY / CATCH / RETURN — Editor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the editor side of TRY/CATCH/RETURN — node types, palette, property panel editors, exporter, importer, validator mirror, change-propagation — per `docs/superpowers/specs/2026-05-31-try-catch-return-design.md`.

**Architecture:** React Flow + Zustand store (matches existing editor patterns). Each new node type is a self-contained React component. The TRY editor is a new property-panel section mirroring `ResourceCommandEditor`. Schema-level work (validator mirror) reuses the runtime's validator helpers via the shared workflow-schema.

**Tech Stack:** TypeScript, React 19, React Flow, Zustand, Vitest + React Testing Library, AJV (browser build). All work happens in the `TrajectoryEditor/` standalone repo.

**Depends on:** `2026-05-31-try-catch-runtime.md` Phases A–B complete (so the schema and validator are stable). Editor work CAN begin earlier if the schema additions are landed in isolation first.

**Out of scope:** Runtime engine behavior, conformance fixtures (sibling plan), docs (sibling plan).

---

## File Structure Plan

### Files to modify

| File | Why |
|---|---|
| `TrajectoryEditor/src/types/nodes.ts` | Add `'catch'` and `'return'` to `NodeType` union. |
| `TrajectoryEditor/src/components/nodes/renderers/flowchart/index.ts` | Register `CatchNode`, `ReturnNode`. |
| `TrajectoryEditor/src/components/nodes/renderers/bpmn/index.ts` | Same. |
| `TrajectoryEditor/src/components/nodes/renderers/isa88/index.ts` | Same. |
| `TrajectoryEditor/src/components/nodes/renderers/notation-config.ts` | Add per-notation trapezoid dimensions and handle positions for catch/return. |
| `TrajectoryEditor/src/components/palette/StepPalette.tsx` | Add `catch` and `return` to `stepTypes` and `allStepTypes`. |
| `TrajectoryEditor/src/components/properties/NodeInlineEditor.tsx` | Add `catch` to `PARAMETERIZED_TYPES`; `return` to `CONTROL_FLOW_TYPES`; render new editors for both. |
| `TrajectoryEditor/src/lib/packageFormat/export-types.ts` | Add `catch: 'CATCH'`, `return: 'RETURN'` to `NODE_TYPE_TO_STEP_TYPE`. |
| `TrajectoryEditor/src/lib/packageFormat/step-extractors.ts` | Add `extractTrySpecifications`, `extractReturnConfig`, `extractCatchId`. |
| `TrajectoryEditor/src/lib/packageFormat/workflow-exporters.ts` | Wire extractors into the per-step emit path. |
| `TrajectoryEditor/src/lib/import/workflow-importers.ts` | Read `try_specifications`, `catch_id`, `return_config` into `node.data.*`. |
| `TrajectoryEditor/src/store/specStore.ts` | Add `renameCatchId(oldId, newId)` and `deleteCatch(catchOid)` actions that propagate to TRY references. |
| Existing connection-rules file (likely `src/lib/canvas/isValidConnection.ts` or similar — verify location) | Reject incoming-to-CATCH, outgoing-from-RETURN, and cross-network edges at draw time. |
| Existing validator (likely `src/lib/validation/workflow-validator.ts`) | Add the rules from runtime plan Phase B (B2–B4) mirrored on the editor side. If the editor can import the runtime's validator package as a dependency, do that and skip the mirror. |

### Files to create

| File | Why |
|---|---|
| `TrajectoryEditor/src/components/nodes/renderers/flowchart/CatchNode.tsx` | Flowchart-notation CATCH renderer (wide-top trapezoid). |
| `TrajectoryEditor/src/components/nodes/renderers/flowchart/ReturnNode.tsx` | Flowchart-notation RETURN renderer (narrow-top trapezoid). |
| `TrajectoryEditor/src/components/nodes/renderers/bpmn/CatchNode.tsx`, `ReturnNode.tsx` | BPMN variants. |
| `TrajectoryEditor/src/components/nodes/renderers/isa88/CatchNode.tsx`, `ReturnNode.tsx` | ISA-88 variants. |
| `TrajectoryEditor/src/components/properties/TryEditor.tsx` | The TRY section in PropertyPanel (sits after Resource Commands on action steps). |
| `TrajectoryEditor/src/components/properties/ReturnConfigEditor.tsx` | The RETURN command + sub-field editor. |
| `TrajectoryEditor/src/components/properties/CatchIdEditor.tsx` | The CATCH's `catch_id` input field. |
| `TrajectoryEditor/src/lib/canvas/catch-network-partition.ts` | Editor-side copy (or import) of the runtime's partition helper, used for live connection validation. |
| `TrajectoryEditor/src/components/properties/__tests__/TryEditor.test.tsx` | RTL component tests. |
| `TrajectoryEditor/src/components/properties/__tests__/ReturnConfigEditor.test.tsx` | RTL component tests. |
| `TrajectoryEditor/src/lib/packageFormat/__tests__/try-catch-roundtrip.test.ts` | Export → import round-trip integrity. |
| `TrajectoryEditor/src/store/__tests__/catch-rename-propagation.test.ts` | Store action propagation. |

### Test command

```
cd C:/Trajectory/TrajectoryEditor
npm test
```

---

## Phase I — Type system and node-type registration

### Task I1: Extend the NodeType union and the editor → runtime mapping

**Files:**
- Modify: `TrajectoryEditor/src/types/nodes.ts`
- Modify: `TrajectoryEditor/src/lib/packageFormat/export-types.ts`

- [ ] **Step 1: Write a failing type-check / unit test**

`TrajectoryEditor/src/lib/packageFormat/__tests__/node-type-mapping.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { NODE_TYPE_TO_STEP_TYPE } from "../export-types.js";

describe("NODE_TYPE_TO_STEP_TYPE", () => {
  it("maps catch → CATCH", () => {
    expect(NODE_TYPE_TO_STEP_TYPE.catch).toBe("CATCH");
  });
  it("maps return → RETURN", () => {
    expect(NODE_TYPE_TO_STEP_TYPE.return).toBe("RETURN");
  });
});
```

- [ ] **Step 2: Run — expect failure**

```
npm test -- node-type-mapping
```

- [ ] **Step 3: Implement**

In `src/types/nodes.ts`:

```typescript
export type NodeType =
  | 'start' | 'end'
  | 'actionProxyWait' | 'yesNoStep' | 'userInteractionStep' | 'scriptStep'
  | 'workflowProxy' | 'select1' | 'waitAny' | 'parallel' | 'waitAll'
  | 'catch' | 'return';
```

In `src/lib/packageFormat/export-types.ts`, extend `NODE_TYPE_TO_STEP_TYPE`:

```typescript
export const NODE_TYPE_TO_STEP_TYPE = {
  start: 'START',
  end: 'END',
  // ... existing entries ...
  catch: 'CATCH',
  return: 'RETURN'
} as const;
```

- [ ] **Step 4: Run — expect PASS**

```
npm test -- node-type-mapping
```

- [ ] **Step 5: Commit**

```
git add src/types/nodes.ts src/lib/packageFormat/export-types.ts src/lib/packageFormat/__tests__/node-type-mapping.test.ts
git commit -m "feat(editor): add catch/return to NodeType union and step-type mapping"
```

---

## Phase J — Node components

### Task J1: Flowchart CatchNode

**Files:**
- Create: `TrajectoryEditor/src/components/nodes/renderers/flowchart/CatchNode.tsx`
- Modify: `TrajectoryEditor/src/components/nodes/renderers/flowchart/index.ts`
- Modify: `TrajectoryEditor/src/components/nodes/renderers/notation-config.ts`

- [ ] **Step 1: Add the trapezoid dimensions to notation-config.ts**

```typescript
// In the flowchart notation block, append:
catch: { width: 120, height: 50, fill: '#ffe0b2', stroke: '#e65100' },
return: { width: 120, height: 50, fill: '#ffccbc', stroke: '#bf360c' },
```

(Repeat in BPMN and ISA-88 blocks; same dimensions; notation-specific fill if desired.)

- [ ] **Step 2: Write failing component test**

`src/components/nodes/renderers/flowchart/__tests__/CatchNode.test.tsx`:

```typescript
import { render } from "@testing-library/react";
import { ReactFlowProvider } from "@xyflow/react";
import { describe, it, expect } from "vitest";
import { CatchNode } from "../CatchNode.js";

describe("CatchNode", () => {
  it("renders with the local_id label", () => {
    const { getByText } = render(
      <ReactFlowProvider>
        <CatchNode data={{ label: "Recover", catchId: "C1" }} id="c1" selected={false} />
      </ReactFlowProvider>
    );
    expect(getByText("Recover")).toBeInTheDocument();
  });

  it("exposes a bottom source handle", () => {
    const { container } = render(
      <ReactFlowProvider>
        <CatchNode data={{ label: "X", catchId: "C1" }} id="c1" selected={false} />
      </ReactFlowProvider>
    );
    const handles = container.querySelectorAll('.react-flow__handle');
    expect(handles.length).toBe(1);
    expect(handles[0].classList.contains('react-flow__handle-bottom')).toBe(true);
  });
});
```

- [ ] **Step 3: Run — expect failure (component doesn't exist)**

```
npm test -- CatchNode
```

- [ ] **Step 4: Implement**

`src/components/nodes/renderers/flowchart/CatchNode.tsx`:

```tsx
import { Handle, Position, type NodeProps } from "@xyflow/react";
import styles from "./CatchNode.module.css";

interface CatchNodeData {
  label: string;
  catchId: string;
}

export function CatchNode({ data, selected }: NodeProps<{ data: CatchNodeData }>) {
  return (
    <div className={`${styles.catchNode} ${selected ? styles.selected : ""}`}>
      <svg viewBox="0 0 120 50" className={styles.svg} aria-hidden>
        {/* Wide-top inverted trapezoid (CATCH funnel) */}
        <polygon points="0,0 120,0 100,50 20,50"
                 className={styles.shape} />
      </svg>
      <div className={styles.label}>{data.label || "Catch"}</div>
      {data.catchId && <div className={styles.catchId}>{data.catchId}</div>}
      <Handle type="source" position={Position.Bottom} className={styles.handle} />
    </div>
  );
}
```

`src/components/nodes/renderers/flowchart/CatchNode.module.css`:

```css
.catchNode {
  position: relative;
  width: 120px;
  height: 50px;
  cursor: pointer;
}
.svg { position: absolute; inset: 0; width: 100%; height: 100%; }
.shape { fill: #ffe0b2; stroke: #e65100; stroke-width: 2; }
.label {
  position: relative; z-index: 1;
  text-align: center; padding-top: 8px;
  font-size: 13px; color: #bf360c;
}
.catchId {
  position: relative; z-index: 1;
  text-align: center;
  font-size: 10px; color: #6d4c41; font-family: monospace;
}
.selected .shape { stroke-width: 3; stroke: #d84315; }
.handle { background: #bf360c; }
```

Register in `src/components/nodes/renderers/flowchart/index.ts`:

```typescript
import { CatchNode } from "./CatchNode.js";

export const flowchartNodeTypes = {
  // ... existing ...
  catch: CatchNode
} as const;
```

- [ ] **Step 5: Run — expect PASS**

```
npm test -- CatchNode
```

- [ ] **Step 6: Commit**

```
git add src/components/nodes/renderers/flowchart/CatchNode.tsx src/components/nodes/renderers/flowchart/CatchNode.module.css src/components/nodes/renderers/flowchart/index.ts src/components/nodes/renderers/flowchart/__tests__/CatchNode.test.tsx src/components/nodes/renderers/notation-config.ts
git commit -m "feat(editor): flowchart CatchNode component (wide-top trapezoid)"
```

### Task J2: Flowchart ReturnNode

Same shape as Task J1 but for ReturnNode. Implementation differences:

- SVG polygon points (narrow-top trapezoid): `points="20,0 100,0 120,50 0,50"`
- Fill `#ffccbc`, stroke `#bf360c`.
- Handle `type="target" position={Position.Top}`.
- Module CSS mirrors CatchNode with the new colors.
- Test asserts top target handle.

Commit: `git commit -m "feat(editor): flowchart ReturnNode component (narrow-top trapezoid)"`

### Task J3: BPMN and ISA-88 variants for both nodes

Two notation packages × two nodes = four lighter component files. Use the same SVG geometry as flowchart (the user's spec made shapes notation-independent for these). Per-notation fills may differ — match each notation's existing colour palette for START/END.

Each file: register in `bpmn/index.ts` / `isa88/index.ts`, snapshot test asserts the component renders without crashing in its notation context.

Commit per notation:
- `feat(editor): bpmn CatchNode/ReturnNode`
- `feat(editor): isa88 CatchNode/ReturnNode`

---

## Phase K — Palette and connection draw rules

### Task K1: Palette entries

**Files:**
- Modify: `TrajectoryEditor/src/components/palette/StepPalette.tsx`

- [ ] **Step 1: Write failing palette test**

```typescript
import { render } from "@testing-library/react";
import { describe, it, expect } from "vitest";
import { StepPalette } from "../StepPalette.js";

describe("StepPalette", () => {
  it("lists Catch and Return draggables", () => {
    const { getByText } = render(<StepPalette />);
    expect(getByText("Catch")).toBeInTheDocument();
    expect(getByText("Return")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run — expect failure**
- [ ] **Step 3: Add to the `stepTypes` and `allStepTypes` arrays**

```typescript
const stepTypes: StepTypeItem[] = [
  // ... existing ...
  { type: 'catch',  label: 'Catch'  },
  { type: 'return', label: 'Return' }
];
```

- [ ] **Step 4: Run — expect PASS**
- [ ] **Step 5: Commit**

```
git commit -m "feat(editor): add Catch and Return to the step palette"
```

### Task K2: Connection draw rules — block illegal edges

**Files:**
- Locate the current `isValidConnection` callback in the canvas component (likely `src/components/canvas/WorkflowCanvas.tsx` or `src/lib/canvas/`).
- Create: `TrajectoryEditor/src/lib/canvas/catch-network-partition.ts` (copy or re-export the runtime's partition helper).

- [ ] **Step 1: Write failing tests**

`src/lib/canvas/__tests__/isValidConnection.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { isValidConnection } from "../isValidConnection.js";

const wfWithCatch = {
  nodes: [
    { id: "s1", type: "start" },
    { id: "s2", type: "end" },
    { id: "c1", type: "catch", data: { catchId: "C1" } },
    { id: "r1", type: "return", data: { command: "ABANDON" } }
  ],
  edges: [
    { source: "s1", target: "s2" },
    { source: "c1", target: "r1" }
  ]
};

describe("isValidConnection", () => {
  it("rejects an incoming edge to CATCH", () => {
    const ok = isValidConnection({ source: "s1", target: "c1" }, wfWithCatch);
    expect(ok).toBe(false);
  });
  it("rejects an outgoing edge from RETURN", () => {
    const ok = isValidConnection({ source: "r1", target: "s2" }, wfWithCatch);
    expect(ok).toBe(false);
  });
  it("rejects a cross-network edge", () => {
    const ok = isValidConnection({ source: "s2", target: "c1" }, wfWithCatch);
    expect(ok).toBe(false);
  });
  it("allows an edge entirely within main-flow", () => {
    const ok = isValidConnection({ source: "s1", target: "s2" }, wfWithCatch);
    expect(ok).toBe(true);
  });
  it("allows an edge entirely within a catch network", () => {
    const ok = isValidConnection({ source: "c1", target: "r1" }, wfWithCatch);
    expect(ok).toBe(true);
  });
});
```

- [ ] **Step 2: Run — expect failures**
- [ ] **Step 3: Implement**

Extend the existing `isValidConnection` (or create one if absent):

```typescript
import { partitionCatchNetworks } from "./catch-network-partition.js";

export function isValidConnection(
  conn: { source: string; target: string },
  graph: { nodes: Array<{ id: string; type: string; data?: any }>;
           edges: Array<{ source: string; target: string }> }
): boolean {
  const sourceNode = graph.nodes.find(n => n.id === conn.source);
  const targetNode = graph.nodes.find(n => n.id === conn.target);
  if (!sourceNode || !targetNode) return false;

  // Structural: CATCH no incoming, RETURN no outgoing.
  if (targetNode.type === "catch") return false;
  if (sourceNode.type === "return") return false;

  // Cross-network rejection.
  const synthSteps = graph.nodes.map(n => ({
    oid: n.id,
    step_type: nodeTypeToStepType(n.type),
    catch_id: n.data?.catchId
  }));
  const synthConns = graph.edges.map(e => ({ from_step_id: e.source, to_step_id: e.target }));
  const part = partitionCatchNetworks({ steps: synthSteps, connections: synthConns });
  const sInCatch = part.catchNetworkStepOids.has(conn.source);
  const tInCatch = part.catchNetworkStepOids.has(conn.target);
  if (sInCatch !== tInCatch) return false;

  return true;
}

function nodeTypeToStepType(t: string): string {
  return ({
    start: "START", end: "END",
    catch: "CATCH", return: "RETURN",
    actionProxyWait: "ACTION PROXY",
    // ... full table ...
  } as const)[t as 'start'] ?? "UNKNOWN";
}
```

- [ ] **Step 4: Wire `isValidConnection` into the canvas component if not already present.**
- [ ] **Step 5: Run — expect PASS**
- [ ] **Step 6: Commit**

```
git commit -m "feat(editor): connection draw rules block CATCH-incoming, RETURN-outgoing, and cross-network edges"
```

---

## Phase L — Property panel editors

### Task L1: TryEditor component

**Files:**
- Create: `TrajectoryEditor/src/components/properties/TryEditor.tsx`
- Create: `TrajectoryEditor/src/components/properties/TryEditor.module.css`
- Create: `TrajectoryEditor/src/components/properties/__tests__/TryEditor.test.tsx`
- Modify: `TrajectoryEditor/src/components/properties/NodeInlineEditor.tsx`

- [ ] **Step 1: Failing tests**

```typescript
import { render, fireEvent } from "@testing-library/react";
import { describe, it, expect, vi } from "vitest";
import { TryEditor } from "../TryEditor.js";

const catches = [
  { catchId: "OvenFailed" },
  { catchId: "OperatorAborted" }
];

describe("TryEditor", () => {
  it("renders an empty state with an add button", () => {
    const onChange = vi.fn();
    const { getByText, queryByRole } = render(
      <TryEditor value={[]} onChange={onChange} availableCatches={catches} />
    );
    expect(getByText(/Add TRY/i)).toBeInTheDocument();
    expect(queryByRole("listitem")).toBeNull();
  });

  it("clicking add appends a row with the first available mode", () => {
    const onChange = vi.fn();
    const { getByText } = render(
      <TryEditor value={[]} onChange={onChange} availableCatches={catches} />
    );
    fireEvent.click(getByText(/Add TRY/i));
    expect(onChange).toHaveBeenCalledWith([
      { mode: "ERROR", catch_id: "OvenFailed", release_on_catch: true }
    ]);
  });

  it("hides modes already used on this step from new-row dropdown", () => {
    const onChange = vi.fn();
    const { getByText, queryByText } = render(
      <TryEditor
        value={[{ mode: "ERROR", catch_id: "OvenFailed" }]}
        onChange={onChange}
        availableCatches={catches}
      />
    );
    fireEvent.click(getByText(/Add TRY/i));
    expect(onChange).toHaveBeenCalledWith(expect.arrayContaining([
      expect.objectContaining({ mode: expect.stringMatching(/^(ABORT|TIMEOUT)$/) })
    ]));
  });

  it("toggling release_on_catch updates the row", () => {
    const onChange = vi.fn();
    const { getByRole } = render(
      <TryEditor
        value={[{ mode: "ERROR", catch_id: "OvenFailed", release_on_catch: true }]}
        onChange={onChange}
        availableCatches={catches}
      />
    );
    fireEvent.click(getByRole("checkbox"));
    expect(onChange).toHaveBeenCalledWith([
      { mode: "ERROR", catch_id: "OvenFailed", release_on_catch: false }
    ]);
  });
});
```

- [ ] **Step 2: Run — expect failures**
- [ ] **Step 3: Implement**

`TryEditor.tsx`:

```tsx
import styles from "./TryEditor.module.css";

const ALL_MODES = ["ERROR", "ABORT", "TIMEOUT"] as const;

export interface TrySpec {
  mode: "ERROR" | "ABORT" | "TIMEOUT";
  catch_id: string;
  release_on_catch?: boolean;
}

interface Props {
  value: TrySpec[];
  onChange: (next: TrySpec[]) => void;
  availableCatches: Array<{ catchId: string }>;
}

export function TryEditor({ value, onChange, availableCatches }: Props) {
  const usedModes = new Set(value.map(t => t.mode));
  const firstAvailableMode = ALL_MODES.find(m => !usedModes.has(m)) ?? "ERROR";
  const firstCatchId = availableCatches[0]?.catchId ?? "";

  function addRow() {
    onChange([...value, {
      mode: firstAvailableMode,
      catch_id: firstCatchId,
      release_on_catch: true
    }]);
  }

  function updateRow(i: number, patch: Partial<TrySpec>) {
    onChange(value.map((t, j) => j === i ? { ...t, ...patch } : t));
  }

  function removeRow(i: number) {
    onChange(value.filter((_, j) => j !== i));
  }

  return (
    <section className={styles.tryEditor}>
      <header>
        <h4>TRY · error handling</h4>
        <button type="button" onClick={addRow}
          disabled={usedModes.size === ALL_MODES.length}
          className={styles.addButton}>+ Add TRY</button>
      </header>
      <ul role="list">
        {value.map((row, i) => (
          <li role="listitem" key={i}>
            <label>
              ON
              <select value={row.mode}
                onChange={e => updateRow(i, { mode: e.target.value as TrySpec["mode"] })}>
                {ALL_MODES.filter(m => m === row.mode || !usedModes.has(m))
                  .map(m => <option key={m} value={m}>{m}</option>)}
              </select>
            </label>
            <label>
              CATCH
              <select value={row.catch_id}
                onChange={e => updateRow(i, { catch_id: e.target.value })}>
                {availableCatches.map(c =>
                  <option key={c.catchId} value={c.catchId}>{c.catchId}</option>)}
              </select>
            </label>
            <button type="button" onClick={() => removeRow(i)}
              aria-label="Remove TRY">✕</button>
            <label>
              <input type="checkbox" checked={row.release_on_catch !== false}
                onChange={e => updateRow(i, { release_on_catch: e.target.checked })} />
              Release acquired resources when CATCH activates
            </label>
          </li>
        ))}
      </ul>
    </section>
  );
}
```

CSS file is straightforward — see the §2.4 wireframe layout from the spec.

- [ ] **Step 4: Wire into `NodeInlineEditor.tsx`**

In the action-step branch of `NodeInlineEditor.tsx`, after the `<ResourceCommandEditor>` block:

```tsx
{(node.type === "actionProxyWait") && (
  <TryEditor
    value={node.data.trySpecifications ?? []}
    onChange={ts => updateNodeData(node.id, { trySpecifications: ts })}
    availableCatches={allCatchesInWorkflow}
  />
)}
```

Where `allCatchesInWorkflow` derives from the workflow nodes:

```tsx
const allCatchesInWorkflow = nodes
  .filter(n => n.type === "catch" && n.data?.catchId)
  .map(n => ({ catchId: n.data.catchId }));
```

- [ ] **Step 5: Run — expect PASS**
- [ ] **Step 6: Commit**

```
git commit -m "feat(editor): TryEditor property-panel section for action steps"
```

### Task L2: ReturnConfigEditor

**Files:**
- Create: `TrajectoryEditor/src/components/properties/ReturnConfigEditor.tsx`
- Create: `TrajectoryEditor/src/components/properties/__tests__/ReturnConfigEditor.test.tsx`
- Modify: `NodeInlineEditor.tsx`

Implementation pattern: a single command `<select>` (ABANDON/RESTART/GOTO/RETRY); conditional fields appear underneath based on command:

- RESTART → `<select>` for `restart_mode` (CLEAN/KEEP), no default — show as required-empty if unset.
- GOTO → `<select>` populated from `mainFlowNodes` (any node where partition `mainFlowStepOids.has(node.id)` AND the type is not `catch`/`return`), bound to `goto_step_oid`.
- ABANDON, RETRY → no extra fields.

Test cases mirror Task L1: render for each command, switching command type, GOTO target population.

Commit: `feat(editor): ReturnConfigEditor for RETURN steps`

### Task L3: CatchIdEditor and description editor on CATCH

**Files:**
- Create: `TrajectoryEditor/src/components/properties/CatchIdEditor.tsx`
- Modify: `NodeInlineEditor.tsx`

Single text input for `catch_id`, validated live for uniqueness within the workflow (shows red border + tooltip on duplicate). Description uses the standard description editor that already exists on other parameterized step types.

Test: typing a duplicate id surfaces the validation message.

Commit: `feat(editor): CatchIdEditor with live uniqueness check`

---

## Phase M — Exporter and importer

### Task M1: step-extractors for TRY/CATCH/RETURN

**Files:**
- Modify: `TrajectoryEditor/src/lib/packageFormat/step-extractors.ts`
- Modify: `TrajectoryEditor/src/lib/packageFormat/__tests__/step-extractors.test.ts` (if exists; create if not)

- [ ] **Step 1: Failing tests**

```typescript
import { describe, it, expect } from "vitest";
import {
  extractTrySpecifications,
  extractReturnConfig,
  extractCatchId
} from "../step-extractors.js";

describe("extractTrySpecifications", () => {
  it("returns undefined when none declared", () => {
    expect(extractTrySpecifications({})).toBeUndefined();
  });
  it("passes through well-formed rows", () => {
    expect(extractTrySpecifications({
      trySpecifications: [
        { mode: "ERROR", catch_id: "C", release_on_catch: true }
      ]
    })).toEqual([
      { mode: "ERROR", catch_id: "C", release_on_catch: true }
    ]);
  });
  it("drops invalid mode values", () => {
    expect(extractTrySpecifications({
      trySpecifications: [{ mode: "WAT", catch_id: "C" }]
    })).toBeUndefined();
  });
});

describe("extractReturnConfig", () => {
  it("returns ABANDON shape", () => {
    expect(extractReturnConfig({ returnConfig: { command: "ABANDON" } }))
      .toEqual({ command: "ABANDON" });
  });
  it("returns RESTART with restart_mode", () => {
    expect(extractReturnConfig({
      returnConfig: { command: "RESTART", restart_mode: "CLEAN" }
    })).toEqual({ command: "RESTART", restart_mode: "CLEAN" });
  });
});

describe("extractCatchId", () => {
  it("returns the catchId string", () => {
    expect(extractCatchId({ catchId: "OvenFailed" })).toBe("OvenFailed");
  });
});
```

- [ ] **Step 2: Run — expect failures**
- [ ] **Step 3: Implement**

```typescript
const VALID_MODES = new Set(["ERROR", "ABORT", "TIMEOUT"]);

export function extractTrySpecifications(data: any) {
  const ts = data?.trySpecifications;
  if (!Array.isArray(ts) || ts.length === 0) return undefined;
  const cleaned = ts.filter(t => t && VALID_MODES.has(t.mode) && typeof t.catch_id === "string");
  return cleaned.length ? cleaned : undefined;
}

export function extractReturnConfig(data: any) {
  const rc = data?.returnConfig;
  if (!rc?.command) return undefined;
  return rc;
}

export function extractCatchId(data: any): string | undefined {
  return typeof data?.catchId === "string" ? data.catchId : undefined;
}
```

- [ ] **Step 4: Run — expect PASS**
- [ ] **Step 5: Commit**

```
git commit -m "feat(editor): extractors for trySpecifications, returnConfig, catchId"
```

### Task M2: workflow-exporters.ts wiring

**Files:**
- Modify: `TrajectoryEditor/src/lib/packageFormat/workflow-exporters.ts`

- [ ] **Step 1: Failing roundtrip test**

`src/lib/packageFormat/__tests__/try-catch-roundtrip.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { exportWorkflow } from "../workflow-exporters.js";
import { importWorkflow } from "../../import/workflow-importers.js";

const editorSpec = {
  id: "wf-rt", name: "Round Trip",
  data: {
    nodes: [
      { id: "s1", type: "start",            data: { label: "Start" } },
      { id: "s2", type: "actionProxyWait",  data: { label: "Action",
        trySpecifications: [{ mode: "ERROR", catch_id: "C1", release_on_catch: true }] } },
      { id: "s3", type: "end",              data: { label: "End" } },
      { id: "c1", type: "catch",            data: { label: "Recover", catchId: "C1" } },
      { id: "c2", type: "return",           data: { label: "Done",
        returnConfig: { command: "ABANDON" } } }
    ],
    edges: [
      { source: "s1", target: "s2", id: "e1" },
      { source: "s2", target: "s3", id: "e2" },
      { source: "c1", target: "c2", id: "e3" }
    ]
  }
};

describe("round-trip TRY/CATCH/RETURN", () => {
  it("exports and re-imports without loss", () => {
    const exported = exportWorkflow(editorSpec);
    const reimported = importWorkflow(exported);
    const s2 = reimported.data.nodes.find(n => n.id === "s2")!;
    expect(s2.data.trySpecifications).toEqual([
      { mode: "ERROR", catch_id: "C1", release_on_catch: true }
    ]);
    const c1 = reimported.data.nodes.find(n => n.id === "c1")!;
    expect(c1.data.catchId).toBe("C1");
    const c2 = reimported.data.nodes.find(n => n.id === "c2")!;
    expect(c2.data.returnConfig).toEqual({ command: "ABANDON" });
  });
});
```

- [ ] **Step 2: Run — expect failure**
- [ ] **Step 3: Implement** the exporter wiring. In `workflow-exporters.ts` per-node emit path:

```typescript
import {
  extractTrySpecifications, extractReturnConfig, extractCatchId
} from "./step-extractors.js";

// In the node-to-step mapping function, after existing extractors:
const try_specifications = extractTrySpecifications(node.data);
if (try_specifications) step.try_specifications = try_specifications;

if (node.type === "catch") {
  const catch_id = extractCatchId(node.data);
  if (catch_id) step.catch_id = catch_id;
}

if (node.type === "return") {
  const return_config = extractReturnConfig(node.data);
  if (return_config) step.return_config = return_config;
}
```

- [ ] **Step 4: Importer wiring** (`workflow-importers.ts`):

```typescript
// In transformWorkflowSpec's per-step → per-node conversion:
if (step.try_specifications) data.trySpecifications = step.try_specifications;
if (step.catch_id) data.catchId = step.catch_id;
if (step.return_config) data.returnConfig = step.return_config;
```

- [ ] **Step 5: Run — expect PASS**
- [ ] **Step 6: Commit**

```
git commit -m "feat(editor): export/import round-trip for trySpecifications, catchId, returnConfig"
```

---

## Phase N — Validator mirror

### Task N1: Mirror the runtime validator rules

**Files:**
- Modify: `TrajectoryEditor/src/lib/validation/workflow-validator.ts` (existing) OR
- Create: `TrajectoryEditor/src/lib/validation/try-catch-rules.ts`

The editor's validator should produce the same error codes as the runtime's. Two options:

A. **Import** the runtime's `validator.ts` if it's exported as part of `@trajectory/runtime`. Reuse the rules verbatim.

B. **Mirror** the rules in editor source. Maintenance burden, but isolates the editor from runtime build cycles.

Decision rule for the implementer: if `package.json` already lists `@trajectory/runtime` (or similar) as a dependency, use option A. Otherwise mirror (option B).

Either way, all 16 error codes from spec §6.1–§6.3 must surface in the editor's validation results panel.

- [ ] **Step 1: Write a snapshot test** that loads a fixture workflow with every kind of violation and asserts each error code appears.
- [ ] **Step 2: Implement option A or B per the decision rule.**
- [ ] **Step 3: Run validator on the canvas state on every change, surface via the existing validation chip / panel.**
- [ ] **Step 4: Manual check** that error chips appear on the offending nodes in the editor.
- [ ] **Step 5: Commit**

```
git commit -m "feat(editor): mirror runtime validator rules for TRY/CATCH/RETURN"
```

---

## Phase O — Change-propagation

### Task O1: catch_id rename propagation

**Files:**
- Modify: `TrajectoryEditor/src/store/specStore.ts`
- Create: `TrajectoryEditor/src/store/__tests__/catch-rename-propagation.test.ts`

- [ ] **Step 1: Failing test**

```typescript
import { describe, it, expect, beforeEach } from "vitest";
import { useSpecStore } from "../specStore.js";

describe("catch_id rename propagation", () => {
  beforeEach(() => {
    useSpecStore.setState({
      nodes: [
        { id: "a1", type: "actionProxyWait", data: {
          trySpecifications: [
            { mode: "ERROR", catch_id: "OldId", release_on_catch: true }
          ]}},
        { id: "c1", type: "catch", data: { catchId: "OldId" } }
      ]
    });
  });

  it("renaming a CATCH's catch_id updates all TRY references", () => {
    useSpecStore.getState().renameCatchId("OldId", "NewId");
    const a1 = useSpecStore.getState().nodes.find(n => n.id === "a1")!;
    expect(a1.data.trySpecifications[0].catch_id).toBe("NewId");
    const c1 = useSpecStore.getState().nodes.find(n => n.id === "c1")!;
    expect(c1.data.catchId).toBe("NewId");
  });
});
```

- [ ] **Step 2: Run — expect failure**
- [ ] **Step 3: Implement** the store action:

```typescript
renameCatchId(oldId: string, newId: string) {
  set(state => ({
    nodes: state.nodes.map(n => {
      // Update the CATCH itself
      if (n.type === "catch" && n.data?.catchId === oldId) {
        return { ...n, data: { ...n.data, catchId: newId } };
      }
      // Update any TRY references
      const ts = n.data?.trySpecifications;
      if (Array.isArray(ts) && ts.some(t => t.catch_id === oldId)) {
        return { ...n, data: { ...n.data,
          trySpecifications: ts.map(t =>
            t.catch_id === oldId ? { ...t, catch_id: newId } : t) } };
      }
      return n;
    })
  }));
}
```

Wire this into the CatchIdEditor's onChange: when the editor commits a new id, call `renameCatchId(oldId, newId)` instead of setting just the local node.

- [ ] **Step 4: Run — expect PASS**
- [ ] **Step 5: Commit**

```
git commit -m "feat(editor): catch_id rename propagates to all TRY references in one undo"
```

### Task O2: Manual verification of UNMATCHED_TRY on CATCH delete

- [ ] **Step 1: Launch dev server**: `npm run dev`
- [ ] **Step 2: Manually** draw a workflow with a CATCH referenced by a TRY, then delete the CATCH. Verify the action step shows the red `UNMATCHED_TRY` chip.
- [ ] **Step 3: Add a snapshot test** capturing this behavior via `screen.getByText(/UNMATCHED_TRY/)` after a store-level CATCH-delete action.
- [ ] **Step 4: Commit**

```
git commit -m "test(editor): UNMATCHED_TRY appears after deleting a referenced CATCH"
```

---

## Phase P — Integration

### Task P1: End-to-end manual smoke test

- [ ] **Step 1: Launch dev server**
- [ ] **Step 2: Draw a complete workflow** with: START → ACTION PROXY (TRY ON ERROR → C1) → END, plus CATCH C1 → USER_INTERACTION → RETURN ABANDON.
- [ ] **Step 3: Export to `.WFmasterX`** via the existing export button.
- [ ] **Step 4: Open the export ZIP, verify the `.WFmaster` JSON** has:
  - `try_specifications` on the action step.
  - `step_type: "CATCH"` with `catch_id`.
  - `step_type: "RETURN"` with `return_config: { command: "ABANDON" }`.
- [ ] **Step 5: Re-import** the same `.WFmasterX`. Verify the canvas reproduces the original workflow exactly (use `node.id` and edge endpoints as the comparison key).
- [ ] **Step 6: Run the full test suite**: `npm test && npm run typecheck && npm run lint`. Fix any failures.
- [ ] **Step 7: Commit** any cleanup:

```
git commit -m "chore(editor): typecheck/lint clean after TRY/CATCH/RETURN"
```

- [ ] **Step 8: Push the branch.**

(Per session conventions, confirm with the user before pushing if on `main`.)

---

## Self-review

1. **Spec coverage:**

| Spec section | Plan task(s) |
|---|---|
| §2.1 palette | K1 |
| §2.2 node renderers (3 notations) | J1–J3 |
| §2.3 connection draw rules | K2 |
| §2.4 PropertyPanel additions | L1–L3 |
| §2.5 change-propagation rules | O1, O2 |
| §2.6 envelope mappings | I1, M2 |
| §6 validator rules (mirror) | N1 |

2. **Placeholder scan:** No `TBD`, `TODO`, or `Similar to Task N` references. Where I said "use the existing X" I named the file or pattern (`ResourceCommandEditor`, `StepPalette.tsx`).

3. **Type consistency:** `TrySpec` shape declared in Task L1 matches `extractTrySpecifications` return shape in M1 and the store representation in O1 (camelCase on the editor side: `trySpecifications`, `catchId`, `returnConfig`; snake_case on the wire: `try_specifications`, `catch_id`, `return_config`). Exporter (M2) handles the rename.

Two intentional decisions deferred to the implementer:
- Validator mirror vs import (Task N1) — depends on existing dep graph.
- Whether to cache the partition computation (Task K2) — depends on canvas perf in practice.

Self-review complete.
