# codex-bench

External CI, benchmark, and Windows build orchestration for `hirsaeki/codex`.

This repository deliberately keeps expensive source builds separate from repeatable verification, benchmark execution, and human-controlled release promotion.

```text
hirsaeki/codex revision
  -> build once
  -> reusable artifact
  -> verify without rebuilding
  -> results / verified marker
  -> human promotion when appropriate
```

## Scope

Product code and regression harnesses remain in `hirsaeki/codex`. This repository owns only build/verify orchestration, result collection, and Windows fork-build promotion.

## apply_patch benchmark workflows

### Build Windows apply_patch artifact

Run `Build Windows apply_patch artifact` with a `hirsaeki/codex` revision (branch, tag, or commit SHA).

The build job compiles once and uploads `windows-apply-patch-bin`, containing:

- `windows_apply_patch_baseline.exe`
- `codex-windows-sandbox-setup.exe`
- `codex-command-runner.exe`
- `provenance.json` with the resolved source SHA and SHA-256 hashes of the binaries

### Verify Windows apply_patch artifact

A successful benchmark build automatically triggers `Verify Windows apply_patch artifact` for that exact build run. The same verify workflow can also be dispatched manually with a previous successful `build_run_id`.

Verification downloads the exact build artifact, validates provenance and binary hashes, prepares a fresh runtime environment, and executes the compiled harness directly without Cargo or source compilation.

If verify/workflow glue fails, fix or rerun the verify workflow against the same build run. Do not rebuild unless the source revision or another binary-affecting input changed.

## Windows fork release workflows

Release distribution is intentionally separate from benchmark execution and is never published automatically.

### Build Windows release artifact

Run `Build Windows release artifact` with a `hirsaeki/codex` revision.

It builds the x64 MSVC release binaries once using the same relevant setup as upstream Windows releases, then uses Codex's canonical package builder to create:

- `codex-package-x86_64-pc-windows-msvc.zip`
- `SHA256SUMS`
- `release-provenance.json`

The canonical ZIP contains `codex.exe`, `codex-code-mode-host.exe`, `codex-command-runner.exe`, `codex-windows-sandbox-setup.exe`, and the normal packaged ripgrep resource.

The v1 fork build is explicitly recorded as **unsigned**. It does not attempt to reproduce OpenAI's Azure Trusted Signing setup.

### Verify Windows release artifact

A successful release build automatically triggers `Verify Windows release artifact`. Verification:

1. downloads the exact release build artifact;
2. validates its build-run provenance and SHA-256 hashes;
3. expands the canonical package and checks its required layout/resources;
4. runs packaged `codex.exe --version` and `codex.exe --help`;
5. uploads `windows-release-verified-<build_run_id>` only after all checks pass.

A previous build can be re-verified manually by `build_run_id` without rebuilding it.

### Publish Windows release

`Publish Windows release` is **manual only**. Human intent is required for every public release.

Inputs are:

- `build_run_id` — the release build to promote;
- `release_tag` — for example `windows-v0.151.0-fork.1`;
- optional `release_name`;
- whether to publish as a prerelease (default: true).

Publication refuses to continue unless a non-expired successful verification marker exists for that build run. It then revalidates provenance and hashes and creates a GitHub Release in this repository using the already-built assets. It never rebuilds source during publication and never overwrites an existing release tag.

Release notes identify the exact `hirsaeki/codex` source SHA and clearly warn that the fork binary is unsigned.

The decision to publish remains human because a successful build/verify means only that the artifact is technically valid; it does not by itself mean that a particular source revision should be distributed.

## Non-goals for v1

This repository does not provide a generic benchmark framework, result database, UI, automatic bisect, persistent binary registry, cross-repository release credentials, ARM64 fork releases, or a custom signing infrastructure. Add those only when a concrete need appears.
