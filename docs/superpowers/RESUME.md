# Resume — TRY/CATCH/RETURN (paused 2026-05-31)

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
