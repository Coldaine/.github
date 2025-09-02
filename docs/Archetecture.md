# Org‑Wide CI Playbook (Rust & Python)

A modular, auto‑adapting CI design you can drop into a `.github` org repo and have it “just work” across all projects. It runs only what’s relevant for each repo and is easy to extend (new languages, extra checks) without touching downstream projects.

---

## TL;DR

1. Add these files to **`Coldaine/.github`** (or your org’s `.github` repo):

   * Composite action: `actions/detect-project/action.yml`
   * Orchestration workflow: `.github/workflows/org-ci.yml`
   * Reusable workflows: `.github/workflows/ci-python.yml`, `.github/workflows/ci-rust.yml`
2. In each project repo, add a **one‑line shim** workflow that **calls** your org workflow (or enforce via Required Workflows):

```yaml
# <project>/.github/workflows/ci.yml
name: Org CI
on:
  pull_request:
  push:
    branches: [main, master]
jobs:
  call-org-ci:
    uses: Coldaine/.github/.github/workflows/org-ci.yml@main
    secrets: inherit
```

The org workflow auto-detects languages/tools and **skips** jobs that don’t apply.

---

## Repository Layout (in `Coldaine/.github`)

```
.github/
  workflows/
    org-ci.yml                # Orchestration (runs everywhere)
    ci-python.yml             # Reusable Python CI (workflow_call)
    ci-rust.yml               # Reusable Rust CI   (workflow_call)
  actions/
    detect-project/
      action.yml              # Composite action: sets capability flags
  workflow-templates/         # (optional) starter templates for repos
```

---

## Composite Action — `actions/detect-project/action.yml`

Detects what’s in a repo (Python/Rust presence, manager, tests, type-check config, Rust workspace/MSRV, pre-commit). Exposes **outputs** you can gate jobs/steps on.

```yaml
name: Detect project languages & tools
description: Detects Python/Rust presence and common tooling; exposes booleans/strings
runs:
  using: "composite"
  steps:
    - id: detect
      shell: bash
      run: |
        set -euo pipefail
        shopt -s nullglob

        # --- Python detection ---
        has_python=false
        for f in pyproject.toml requirements.txt setup.cfg setup.py Pipfile poetry.lock uv.lock; do
          [[ -f "$f" ]] && has_python=true && break
        done
        echo "has_python=$has_python" >> "$GITHUB_OUTPUT"

        python_manager=auto
        if [[ -f poetry.lock ]] || grep -q '^\[tool\.poetry\]' pyproject.toml 2>/dev/null; then
          python_manager=poetry
        elif [[ -f uv.lock ]]; then
          python_manager=uv
        elif [[ -f requirements.txt ]] || [[ -f setup.cfg ]] || [[ -f setup.py ]]; then
          python_manager=pip
        fi
        echo "python_manager=$python_manager" >> "$GITHUB_OUTPUT"

        has_pytests=false
        if compgen -G "tests/**/*.py" > /dev/null || compgen -G "test/**/*.py" > /dev/null || grep -q 'pytest' pyproject.toml 2>/dev/null; then
          has_pytests=true
        fi
        echo "has_pytests=$has_pytests" >> "$GITHUB_OUTPUT"

        has_mypy=false
        if [[ -f mypy.ini ]] || grep -q '^\[tool\.mypy\]' pyproject.toml 2>/dev/null; then
          has_mypy=true
        fi
        echo "has_mypy=$has_mypy" >> "$GITHUB_OUTPUT"

        has_precommit=false
        if [[ -f .pre-commit-config.yaml ]] || [[ -f .pre-commit-config.yml ]]; then
          has_precommit=true
        fi
        echo "has_precommit=$has_precommit" >> "$GITHUB_OUTPUT"

        # --- Rust detection ---
        has_rust=false
        if [[ -f Cargo.toml ]] || [[ -f rust-toolchain.toml ]]; then
          has_rust=true
        fi
        echo "has_rust=$has_rust" >> "$GITHUB_OUTPUT"

        rust_msrv=$(grep -m1 -Po '(?<=^rust-version\s*=\s*")[^"]+' Cargo.toml 2>/dev/null || true)
        echo "rust_msrv=$rust_msrv" >> "$GITHUB_OUTPUT"

        rust_workspace=false
        if grep -q '^\[workspace\]' Cargo.toml 2>/dev/null; then
          rust_workspace=true
        fi
        echo "rust_workspace=$rust_workspace" >> "$GITHUB_OUTPUT"
outputs:
  has_python:      { description: "true/false" }
  has_rust:        { description: "true/false" }
  python_manager:  { description: "pip | poetry | uv | auto" }
  has_pytests:     { description: "true/false" }
  has_mypy:        { description: "true/false" }
  has_precommit:   { description: "true/false" }
  rust_msrv:       { description: "MSRV from Cargo.toml if set" }
  rust_workspace:  { description: "true/false" }
```

> **Why**: keep “what should we run?” logic in one maintained place. The rest of your CI only reads these outputs.

---

## Orchestration Workflow — `.github/workflows/org-ci.yml`

Cancels redundant runs, short-circuits on docs‑only changes, then **calls** language workflows only if they apply.

```yaml
name: CI (org defaults)

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
  push:
    branches: [main, master]
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: read

concurrency:
  group: ci-${{ github.ref }}-${{ github.workflow }}
  cancel-in-progress: true

jobs:
  detect:
    runs-on: ubuntu-latest
    outputs:
      has_python:      ${{ steps.detect.outputs.has_python }}
      has_rust:        ${{ steps.detect.outputs.has_rust }}
      python_manager:  ${{ steps.detect.outputs.python_manager }}
      has_pytests:     ${{ steps.detect.outputs.has_pytests }}
      has_mypy:        ${{ steps.detect.outputs.has_mypy }}
      has_precommit:   ${{ steps.detect.outputs.has_precommit }}
      rust_msrv:       ${{ steps.detect.outputs.rust_msrv }}
      rust_workspace:  ${{ steps.detect.outputs.rust_workspace }}
      docs_only:       ${{ steps.filter.outputs.docs_only }}
    steps:
      - uses: actions/checkout@v4
      - id: filter
        uses: dorny/paths-filter@v3
        with:
          filters: |
            docs_only:
              - 'docs/**'
              - '**/*.md'
              - '!pyproject.toml'
              - '!Cargo.toml'
              - '!**/*.rs'
              - '!**/*.py'
      - id: detect
        uses: Coldaine/.github/actions/detect-project@main

  python:
    needs: detect
    if: needs.detect.outputs.has_python == 'true' && needs.detect.outputs.docs_only != 'true'
    uses: Coldaine/.github/.github/workflows/ci-python.yml@main
    secrets: inherit
    with:
      python-versions: '["3.10","3.11","3.12","3.13"]'
      os: '["ubuntu-latest"]'
      package-manager: "auto"      # pip|poetry|uv|auto
      run-tests:      ${{ needs.detect.outputs.has_pytests == 'true' }}
      run-type-check: ${{ needs.detect.outputs.has_mypy == 'true' }}
      run-precommit:  ${{ needs.detect.outputs.has_precommit == 'true' }}

  rust:
    needs: detect
    if: needs.detect.outputs.has_rust == 'true' && needs.detect.outputs.docs_only != 'true'
    uses: Coldaine/.github/.github/workflows/ci-rust.yml@main
    secrets: inherit
    with:
      toolchains: '["stable"]'     # add "beta","nightly" as desired
      os: '["ubuntu-latest","macos-latest","windows-latest"]'
      msrv: ${{ needs.detect.outputs.rust_msrv }}
      deny-warnings: true
      run-coverage: true
```

> **Why**: cheap path filtering makes README‑only changes skip compute‑heavy jobs; language workflows run only when the detector says they’re needed.

---

## Reusable Python CI — `.github/workflows/ci-python.yml`

* Matrix over Python versions & OS
* Detects and uses **pip / poetry / uv**
* Optional **pre‑commit**, **ruff/black/isort**, **mypy**, **pytest**, **pip‑audit**, optional **Bandit**
* Optional **Codecov** coverage upload

```yaml
name: Reusable Python CI

on:
  workflow_call:
    inputs:
      python-versions:
        description: 'JSON array of versions'
        required: false
        type: string
        default: '["3.10","3.11","3.12","3.13"]'
      os:
        description: 'JSON array of runners'
        required: false
        type: string
        default: '["ubuntu-latest"]'
      package-manager:
        description: 'auto|pip|poetry|uv'
        required: false
        type: string
        default: 'auto'
      run-tests:
        type: boolean
        default: true
      run-type-check:
        type: boolean
        default: false
      run-precommit:
        type: boolean
        default: false

jobs:
  py:
    name: "Python (${{ matrix.python }}) on ${{ matrix.os }}"
    runs-on: ${{ matrix.os }}
    permissions:
      contents: read
      pull-requests: read
    strategy:
      fail-fast: false
      matrix:
        python: ${{ fromJson(inputs["python-versions"]) }}
        os:     ${{ fromJson(inputs.os) }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python }}
        id: setup
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python }}
          cache: ${{ (inputs.package-manager == 'pip' || inputs.package-manager == 'auto') && 'pip' || '' }}

      - name: Detect Python packaging (if auto)
        if: inputs.package-manager == 'auto'
        id: pm
        shell: bash
        run: |
          set -euo pipefail
          pm=pip
          if [[ -f poetry.lock ]] || grep -q '^\[tool\.poetry\]' pyproject.toml 2>/dev/null; then pm=poetry
          elif [[ -f uv.lock ]]; then pm=uv
          fi
          echo "pm=$pm" >> "$GITHUB_OUTPUT"

      - name: Normalize PM
        id: norm
        run: echo "pm=${{ inputs.package-manager == 'auto' && steps.pm.outputs.pm || inputs.package-manager }}" >> "$GITHUB_OUTPUT"

      - name: Install deps (pip)
        if: steps.norm.outputs.pm == 'pip'
        run: |
          python -m pip install --upgrade pip
          if [[ -f requirements.txt ]]; then pip install -r requirements.txt; fi
          if [[ -f pyproject.toml && ! -f requirements.txt ]]; then pip install -e ".[dev,test]" || true; fi

      - name: Install deps (poetry)
        if: steps.norm.outputs.pm == 'poetry'
        run: |
          python -m pip install poetry
          poetry install --no-interaction --no-root || poetry install --no-interaction

      - name: Install deps (uv)
        if: steps.norm.outputs.pm == 'uv'
        run: |
          python -m pip install uv
          if [[ -f uv.lock ]]; then uv sync --frozen; else uv pip install -r requirements.txt; fi

      - name: Pre-commit
        if: inputs.run-precommit
        uses: pre-commit/action@v3.0.1

      - name: Lint (ruff) + formatting checks
        run: |
          python -m pip install ruff black isort || true
          ruff check .
          black --check .
          isort --check-only .

      - name: Type-check (mypy)
        if: inputs.run-type-check
        run: |
          python -m pip install mypy || true
          mypy --install-types --non-interactive || true
          mypy .

      - name: Unit tests (pytest)
        if: inputs.run-tests
        run: |
          python -m pip install pytest pytest-cov pytest-xdist || true
          pytest -q -n auto --maxfail=1 --disable-warnings \
            --junitxml=test-results/junit.xml \
            --cov=. --cov-report=xml

      - name: Upload test results
        if: inputs.run-tests && (success() || failure())
        uses: actions/upload-artifact@v4
        with:
          name: junit-python-${{ matrix.python }}-${{ matrix.os }}
          path: test-results/junit.xml

      - name: Security (pip-audit)
        run: |
          python -m pip install pip-audit || true
          if [[ -f requirements.txt ]]; then pip-audit -r requirements.txt || true; else pip-audit || true; fi

      - name: Bandit (optional)
        if: hashFiles('**/.bandit','**/.bandit*') != ''
        uses: PyCQA/bandit-action@v1

      - name: Upload coverage to Codecov (optional)
        if: inputs.run-tests && (env.CODECOV_TOKEN != '' || github.repository_owner == 'codecov')
        uses: codecov/codecov-action@v4
        with:
          fail_ci_if_error: false
```

**Notes**

* `actions/setup-python` supports built‑in caching for `pip`. For `poetry`/`uv`, leverage their own caches/lockfiles.
* To merge multi‑job coverage, you can add a final “combine” job that downloads artifacts and runs `coverage combine`.

---

## Reusable Rust CI — `.github/workflows/ci-rust.yml`

* Matrix over toolchains & OS
* `dtolnay/rust-toolchain` for concise installs, `Swatinem/rust-cache` for smart caching
* fmt / clippy / tests; optional **nextest**
* Optional **coverage** via `cargo-llvm-cov` with LCOV upload to Codecov
* Optional **supply chain** checks (`cargo-audit`, `cargo-deny`)

```yaml
name: Reusable Rust CI

on:
  workflow_call:
    inputs:
      toolchains:
        description: 'JSON array (e.g. ["stable","beta"])'
        required: false
        type: string
        default: '["stable"]'
      os:
        description: 'JSON array of runners'
        required: false
        type: string
        default: '["ubuntu-latest"]'
      msrv:
        description: 'Minimum Supported Rust Version (e.g. "1.74")'
        required: false
        type: string
        default: ''
      deny-warnings:
        type: boolean
        default: true
      run-coverage:
        type: boolean
        default: false

jobs:
  build-test:
    name: "Rust (${{ matrix.toolchain }}) on ${{ matrix.os }}"
    runs-on: ${{ matrix.os }}
    permissions:
      contents: read
      pull-requests: read
    strategy:
      fail-fast: false
      matrix:
        toolchain: ${{ fromJson(inputs.toolchains) }}
        os: ${{ fromJson(inputs.os) }}

    steps:
      - uses: actions/checkout@v4

      - name: Install Rust (${{ matrix.toolchain }})
        uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ matrix.toolchain }}
          components: clippy, rustfmt

      - name: Cache cargo/target
        uses: Swatinem/rust-cache@v2

      - name: rustfmt
        run: cargo fmt --all -- --check

      - name: clippy
        run: cargo clippy --all-targets --all-features ${{ inputs.deny-warnings && ' -- -D warnings' || '' }}

      - name: Tests
        run: cargo test --workspace --all-features --all-targets

      - name: Coverage (llvm-cov -> lcov)
        if: inputs.run-coverage && runner.os == 'Linux'
        run: |
          cargo install cargo-llvm-cov || true
          cargo llvm-cov --workspace --all-features --lcov --output-path lcov.info

      - name: Upload coverage to Codecov
        if: inputs.run-coverage && runner.os == 'Linux'
        uses: codecov/codecov-action@v4
        with:
          files: lcov.info
          fail_ci_if_error: false

  msrv:
    if: inputs.msrv != ''
    runs-on: ubuntu-latest
    permissions: { contents: read }
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with: { toolchain: ${{ inputs.msrv }} }
      - uses: Swatinem/rust-cache@v2
      - run: cargo build --workspace --all-features

  supply-chain:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - name: cargo-audit (advisories)
        uses: actions-rust-lang/audit@v1
        with: { denyWarnings: false }
      - name: cargo-deny (licenses/bans/sources)
        uses: EmbarkStudios/cargo-deny-action@v2
```

**Notes**

* Add cross‑compilation targets or `sccache` if needed.
* `deny-warnings: true` keeps clippy strict; expose as input so repos can loosen if necessary.

---

## Smart‑Skip & Cost Controls

* **Path filtering** (in `org-ci.yml`) sets `docs_only=true` to bypass compute‑heavy jobs for README/docs changes.
* **Detector outputs** turn entire lanes on/off before jobs even start.
* **Concurrency** cancels superseded runs on the same branch/ref.

---

## Security & Hygiene

* **Pin action versions** (ideally to SHAs) and set **least‑privilege** `permissions:` at workflow/job level.
* Prefer **OIDC** for publishing (e.g., PyPI) over long‑lived tokens.
* Add optional security jobs: `pip-audit`, `Bandit`, `cargo-audit`, `cargo-deny`.

---

## Extensibility Patterns

Add another language in three steps:

1. **Detect**: extend `detect-project` to set `has_node`/`has_go` etc.
2. **Reusable workflow**: create `ci-node.yml` / `ci-go.yml` with inputs.
3. **Orchestrate**: add a job to `org-ci.yml` gated on the new `has_*` flag (+ optional paths‑filter entries).

This keeps repos hands‑off: they continue to call only the org workflow.

---

## Optional Tri‑State Toggles (`auto|true|false`)

If you prefer explicit controls, model inputs like:

```yaml
# example: in org-ci.yml when calling a lane
with:
  run-python: "auto"   # auto|true|false
```

Then in job `if:` conditions, run when:

```
(run-python == 'true') || (run-python == 'auto' && has_python == 'true')
```

Same idea for `run-rust`.

---

## Troubleshooting

* **Python not detected**: ensure one of `pyproject.toml`, `requirements.txt`, `setup.cfg`, `setup.py`, `poetry.lock`, or `uv.lock` exists.
* **Rust not detected**: ensure there’s a `Cargo.toml` (workspace root for workspaces).
* **Codecov**: uploads are optional; if missing token on private repos, step will no‑op if configured as shown.

---

## Changelog (fill in as you adopt)

* v1: Initial org‑wide modular CI (detector + orchestrator + Python/Rust lanes)
* v1.1: Add docs‑only short‑circuiting and tri‑state toggles
* v1.2: Enable MSRV job and supply‑chain checks by default

---

**That’s it.** Drop these into your `.github` repo, add the one‑line shim in projects (or use Required Workflows), and you’ve got centralized, adaptive CI for Rust & Python with room to grow.
