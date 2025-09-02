# Organization-wide CI: Coldaine/.github

This repo hosts a reusable, multi-language CI that your repos can call with a single line. It routes to Python and/or Rust lanes automatically and smart-skips when irrelevant files change.

## What you get

- Path-based language detection to skip unused lanes
- Python: ruff, mypy, pytest (xdist), coverage, Bandit, optional pip-audit
- Rust: fmt, clippy, build, test (nextest), coverage (llvm-cov), cargo-deny, optional cargo-audit
- Caching: pip and cargo (with optional sccache)
- Least-privilege permissions + fork-safe Codecov uploads
- Per-lane concurrency so Python and Rust runs don’t cancel each other

## Files

- `.github/workflows/lang-ci.yml` — Router workflow (call this from your repos)
- `.github/workflows/reusable/python-ci.yml` — Python lane
- `.github/workflows/reusable/rust-ci.yml` — Rust lane

## How to use

Recommended: pin to a stable tag (v1).

- `.github/workflows/ci.yml`
- name: CI
  on:
    pull_request:
    push:
  jobs:
    ci:
      uses: Coldaine/.github/.github/workflows/lang-ci.yml@v1

That’s it. The router will run only the lanes that matter based on changed files.

### Optional inputs

You can override behavior by passing inputs. Examples:

- `.github/workflows/ci.yml`
- name: CI
  on:
    pull_request:
    push:
  jobs:
    ci:
      uses: Coldaine/.github/.github/workflows/lang-ci.yml@v1
      with:
        python-versions: '["3.11","3.12"]'
        pkg-manager: auto          # auto|uv|pip|poetry|pdm
        rust-channel: stable       # stable|nightly|1.75.0
        rust-use-nextest: true
        rust-coverage: true
        rust-use-sccache: false

All boolean run controls default to true; path filters will prevent unnecessary lanes from running.

## Coverage uploads

- Public repos: tokenless Codecov upload works.
- Forked PRs: uploads are skipped to avoid secret leakage.
- Private repos: define a CODECOV_TOKEN secret at org or repo level.

## Security and hygiene

- Actions pinned to specific SHAs where possible.
- contents: read permissions by default.
- cargo-deny and cargo-audit enabled by default (can be disabled via inputs).
- Bandit and pip-audit enabled by default for Python.

## Versioning

Use `@v1` for stability. We’ll update `v1` with backward-compatible improvements. For breaking changes we’ll publish `v2`.

## Notes

- If your repo layout is unusual (nested workspaces, monorepo), adjust inputs like `rust-workspace-members`.
- If a repo has no Python/Rust changes, the corresponding lane will be skipped automatically.
