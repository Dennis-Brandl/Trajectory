# TRY / CATCH / RETURN — Documentation Update Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update the public-interface docs to reflect the new TRY/CATCH/RETURN feature, and add a runtime-implementer reference doc.

**Architecture:** Three independent doc edits. Each commits separately. No code change. No tests beyond markdown link/heading verification.

**Tech Stack:** Markdown. Run in the umbrella `Trajectory/` super-repo for docs/JSON_SCHEMA.md and docs/REST_Protocol.md; run in `TrajectoryRuntime/` for the new error-handling.md.

**Depends on:** The runtime and editor plans being merged (or at least the schema decisions being final).

---

## File Structure Plan

### Files to modify

| File | Repo | Change |
|---|---|---|
| `Trajectory/docs/JSON_SCHEMA.md` | umbrella | Extend step_type enum (§1.5.1); add `try_specifications` to per-step fields (§1.5.7); add CATCH config and RETURN config subsections; add new node-type mapping rows in Part 4. |
| `Trajectory/docs/REST_Protocol.md` | umbrella | Add one paragraph in §3.3 about runtime-side failure classification (informational). |

### Files to create

| File | Repo | Purpose |
|---|---|---|
| `TrajectoryRuntime/spec/docs/error-handling.md` | runtime | Reference doc for runtime implementers. Source: the approved spec at `Trajectory/docs/superpowers/specs/2026-05-31-try-catch-return-design.md` — condensed to runtime-only concerns. |

### Test command

```
# Umbrella (for the two existing doc edits)
cd C:/Trajectory/Trajectory
# Manually verify rendered output in any markdown viewer.

# Runtime
cd C:/Trajectory/TrajectoryRuntime
# Same — markdown, no automated tests.
```

---

## Phase Q — Docs

### Task Q1: Extend `docs/JSON_SCHEMA.md` with the new step types and configs

**Files:**
- Modify: `Trajectory/docs/JSON_SCHEMA.md`

- [ ] **Step 1: Add CATCH and RETURN to §1.5.1 step type enum table**

Locate the §1.5.1 table. Append two rows:

```markdown
| `"CATCH"` | Catch network entry; auto-completes after writing trigger info. See §1.5.9. | Yes |
| `"RETURN"` | Catch network exit; dispatches ABANDON/RESTART/GOTO/RETRY. See §1.5.10. | Yes |
```

- [ ] **Step 2: Add `try_specifications` to §1.5.7 (per-step parameter & resource arrays)**

Append a row to the §1.5.7 table:

```markdown
| `try_specifications` | array of `{mode, catch_id, release_on_catch?}` — error-handler bindings. Valid only on `ACTION PROXY` / `WAIT ACTION PROXY`. See §1.5.11. |
```

- [ ] **Step 3: Add three new subsections**

After the existing §1.5.8 (form-element image registry), add §1.5.9 (CATCH config), §1.5.10 (RETURN config), §1.5.11 (TRY specification), each containing the JSON shape and field rules from the design spec §1.

- [ ] **Step 4: Extend Part 4 (editor → runtime envelope) with the new mapping rows**

Add to the existing node-type table:

```markdown
| `catch` | `CATCH` |
| `return` | `RETURN` |
```

Add to the existing `node.data.*` mapping table:

```markdown
| `node.data.trySpecifications` | `try_specifications` |
| `node.data.catchId` | `catch_id` |
| `node.data.returnConfig` | `return_config` |
```

- [ ] **Step 5: Visually verify the file** — open in any markdown viewer; confirm all four new sections render cleanly, all tables align.
- [ ] **Step 6: Commit**

```
cd C:/Trajectory/Trajectory
git add docs/JSON_SCHEMA.md
git commit -m "docs(schema): add TRY/CATCH/RETURN coverage to JSON_SCHEMA.md"
```

### Task Q2: One-paragraph note in `docs/REST_Protocol.md` §3.3

**Files:**
- Modify: `Trajectory/docs/REST_Protocol.md`

- [ ] **Step 1: Locate §3.3** (opaque vs observable).
- [ ] **Step 2: Add the paragraph**

After the existing §3.3 content, append:

```markdown
### 3.3.1 Runtime-side failure classification (informational)

The Trajectory Runtime classifies AC terminal events into one of three failure modes — `ERROR`, `ABORT`, `TIMEOUT` — for TRY/CATCH dispatch (see `error-handling.md` in the TrajectoryRuntime spec). This classification is entirely runtime-side and does NOT require any Action Container protocol change. The runtime uses:

- A wall-clock timer started at REST-03 invoke (`timeout_ms` from request, or the action's `timeout_seconds`). On expiry, the runtime sends `ABORT` to the AC and records a TIMEOUT marker.
- The AC's terminal `state.current` and `error` fields, plus the runtime's own record of any commands it sent to this instance, to distinguish `ERROR` (action_code failure) from `ABORT` (runtime-initiated termination).

The AC continues to expose the same wire format as documented above; no AC implementation work is needed to support TRY/CATCH/RETURN.
```

- [ ] **Step 3: Visually verify.**
- [ ] **Step 4: Commit**

```
git add docs/REST_Protocol.md
git commit -m "docs(rest): note runtime-side failure classification for TRY/CATCH dispatch"
```

### Task Q3: New `spec/docs/error-handling.md` in the runtime repo

**Files:**
- Create: `TrajectoryRuntime/spec/docs/error-handling.md`

- [ ] **Step 1: Write the doc**

Structure mirroring the existing `RESTProtocolSpec.md`:

```markdown
# Trajectory Runtime — Error Handling Reference

**Audience:** Runtime implementers.
**Source:** Distilled from the approved design at `Trajectory/docs/superpowers/specs/2026-05-31-try-catch-return-design.md`.

## 1. Concepts
[Brief: TRY, CATCH, RETURN; strict-island topology; failure modes.]

## 2. Schema reference
[Reproduce the `TrySpecification`, CATCH, and RETURN shapes from the design spec §1. Cross-link to `Trajectory/docs/JSON_SCHEMA.md`.]

## 3. State machine
[Adapt design spec §3 — CATCH/RETURN auto-complete, `active_catches` map, concurrency under PARALLEL.]

## 4. Failure classification
[Adapt design spec §4 — wall-clock timer, classification table, TRY dispatch pseudocode, user-abort interaction.]

## 5. RETURN command dispatch
[Adapt design spec §5 — all four commands, summary table.]

## 6. Validator rule catalog
[Adapt design spec §6 — error code inventory with severities. Cross-link to validator source.]

## 7. Test fixture index
[Link to each fixture under spec/conformance/execution/exec-try-catch-*.json and spec/conformance/validation/valid-*.json, with a one-line summary of what each asserts.]

## Appendix A — File touch points
[Copy the runtime portion of design spec Appendix B.]
```

Length target: 400–600 lines. Tone: implementer reference, not narrative.

- [ ] **Step 2: Cross-check** that every error code in §6 matches the runtime's `validator.ts` and the test fixtures.
- [ ] **Step 3: Commit**

```
cd C:/Trajectory/TrajectoryRuntime
git add spec/docs/error-handling.md
git commit -m "docs(runtime): error-handling reference for TRY/CATCH/RETURN implementers"
```

---

## Self-review

1. **Spec coverage:** §7.5 of the design spec enumerates exactly three doc deltas — all three are addressed (Q1, Q2, Q3).

2. **Placeholder scan:** Q3's task body deliberately delegates structure (Section list + length target) rather than dumping content — but the content source is the already-committed design spec, so there's no placeholder; the implementer copies from a known location. Q1 and Q2 have full content blocks.

3. **No code change.** No `npm test` to run. Verification is visual.

Self-review complete.
