# codex-bench

External CI/benchmark orchestration for `hirsaeki/codex`.

This repository deliberately keeps expensive source builds separate from repeatable verification and benchmark execution.

```text
hirsaeki/codex revision
  -> build once
  -> reusable Windows artifact
  -> prepare runtime environment
  -> execute compiled harness directly
  -> result artifact
```

## Scope

The initial surface is intentionally narrow: native Windows `apply_patch` verification using the regression harness that lives in `hirsaeki/codex` at `codex-rs/core/tests/windows_apply_patch_baseline.rs`.

Product code and the regression harness remain in `hirsaeki/codex`. This repository owns only build/verify orchestration and result collection.

## Workflows

### Build Windows apply_patch artifact

Run `Build Windows apply_patch artifact` with a `hirsaeki/codex` revision (branch, tag, or commit SHA).

The build job compiles once and uploads `windows-apply-patch-bin`, containing:

- `windows_apply_patch_baseline.exe`
- `codex-windows-sandbox-setup.exe`
- `codex-command-runner.exe`
- `provenance.json` with the resolved source SHA and SHA-256 hashes of the binaries

The artifact is scoped to that workflow run, so its name can stay constant.

### Verify Windows apply_patch artifact

Run `Verify Windows apply_patch artifact` with only the successful build workflow's `build_run_id`.

The verify job:

1. downloads the exact build artifact from that run;
2. validates binary hashes against `provenance.json`;
3. creates a fresh `CODEX_HOME` and puts the staged helpers on `PATH`;
4. executes `windows_apply_patch_baseline.exe` directly, without Cargo or source compilation;
5. uploads the raw harness log, sandbox logs, parsed JSON results, and verify metadata.

If verify/workflow glue fails, fix or rerun the verify workflow against the same build run. Do not rebuild unless the source revision or another binary-affecting input changed.

## Non-goals for v1

This repository does not yet provide a generic benchmark framework, result database, UI, automatic bisect, persistent binary registry, or generalized A/B runner. Add those only when a concrete need appears.
