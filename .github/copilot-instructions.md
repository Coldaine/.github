# CI Router: Quick Guide for Copilots

- Big picture: `.github/workflows/lang-ci.yml` is a router. It detects Python/Rust changes and delegates to reusable lanes in `./.github/workflows/reusable/`.
- Events: triggers on `push`, `pull_request`, and `workflow_dispatch` (with ≤10 inputs), plus `workflow_call` so other repos can reuse it.
- Smart skip: `dorny/paths-filter@v3` gates lanes so we don’t run Python jobs on Rust-only changes (and vice versa).
- Concurrency: per-lane groups cancel in-flight runs for the same ref to save CI.
- Permissions: least-privilege (`contents: read`, `pull-requests: read`). All actions are pinned to SHAs.

## Python lane (reusable/python-ci.yml)
- Package manager: auto-detects `uv`, `poetry`, `pdm`, or `pip`.
- Lint: `ruff` (format/lint). Typecheck: `mypy`.
- Test: `pytest` with coverage; coverage artifact uploaded; Codecov upload is gated for forks.
- Security: `bandit` enabled; optional `pip-audit` (config-driven).
- Inputs (via router): versions, pkg-manager, lint/typecheck toggles, coverage on/off, security-check toggle.

## Rust lane (reusable/rust-ci.yml)
- Quality: `cargo fmt --check` and `clippy` (warnings as errors by default).
- Build + Test: `cargo build` and `cargo test` (optional `cargo-nextest`).
- Coverage: `cargo-llvm-cov` with summary artifact and optional upload.
- Supply chain: `cargo-deny` and RustSec audit.
- Options: `msrv`, `features`, `no-default-features`, `workspace-members`, `sccache`, optional ALSA/apt deps.

## Events and inputs
- `push`, `pull_request`: run with sensible defaults; lanes run only if matching files changed.
- `workflow_dispatch`: manual with limited inputs (≤10) to keep UX simple.
- `workflow_call`: allows cross-repo reuse with minimal wiring.

## Cross-repo usage
- Recommend pinning to a tag:
  ```yaml
  jobs:
    ci:
      uses: <owner>/.github/.github/workflows/lang-ci.yml@v1
      with:
        python-versions: '["3.10","3.11","3.12"]'
        rust-channel: stable
  ```

## Manual runs
- From the Actions tab, choose Lang CI (router) → Run workflow. Defaults favor enabled lanes; path gating prevents no-op runs.
