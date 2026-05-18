# Trajectory

A family of related projects for visual workflow specification, execution, and management.

## Sub-projects

| Repo | Purpose |
| --- | --- |
| [TrajectoryEditor](https://github.com/Dennis-Brandl/TrajectoryEditor) | Visual workflow specification editor |
| [TrajectoryRuntime](https://github.com/Dennis-Brandl/TrajectoryRuntime) | Runtime engine that executes specifications |
| [TrajectoryActions](https://github.com/Dennis-Brandl/TrajectoryActions) | Action library used by workflows |
| [TrajectoryActionTester](https://github.com/Dennis-Brandl/TrajectoryActionTester) | Test harness for actions |
| [TrajectoryManager](https://github.com/Dennis-Brandl/TrajectoryManager) | Orchestration / management tooling |

## Cloning

This repo uses git submodules. To get everything in one go:

```sh
git clone --recurse-submodules https://github.com/Dennis-Brandl/Trajectory.git
```

If you already cloned without `--recurse-submodules`:

```sh
git submodule update --init --recursive
```

To pull the latest commit on each submodule's default branch:

```sh
git submodule update --remote --merge
```

## License

[Apache 2.0](LICENSE) &copy; 2026 Dennis Brandl.
