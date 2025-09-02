# Organization-Level Reusable CI Workflow

**Repository:** [Coldaine/.github](https://github.com/Coldaine/.github)  
**Last Updated:** 2025-09-02  
**Status:** ✅ Implemented  
**Version:** v1.0.0  

## Project Summary

This repository provides a centralized, reusable GitHub Actions workflow that standardizes continuous integration (CI) practices across all organization repositories. The workflow supports both Rust and Python projects with extensive configuration options while maintaining secure, performant defaults.

## Key Features

### 🚀 Multi-Language Support
- **Rust:** Complete toolchain with formatting, clippy, testing, documentation, and MSRV checks
- **Python:** Comprehensive testing with ruff, pytest, mypy, and security scanning
- **Auto-detection:** Automatically detects project type based on file presence

### 🔧 Advanced Rust Capabilities
- **Workspace-aware:** Automatically detects and tests all workspace members in parallel
- **Per-crate system dependencies:** Map specific crates to their required system packages
- **Feature testing:** Support for feature flags and no-default-features builds
- **Performance tools:** Optional cargo-nextest and sccache integration
- **Security:** cargo-deny for license compliance and vulnerability scanning

### 🐍 Python Integration
- **Multiple package formats:** Supports pyproject.toml, requirements.txt, and setup.py
- **Modern tooling:** Uses ruff for fast formatting and linting
- **Security scanning:** Includes bandit for security vulnerability detection
- **Coverage reporting:** Built-in pytest coverage reports

### 🔒 Security Best Practices
- **Action pinning:** All actions pinned to SHA with version comments (2025 standard)
- **Minimal permissions:** Read-only by default
- **Supply chain security:** Trivy scanning for dependency vulnerabilities
- **Audit checks:** RustSec advisory database integration

### ⚡ Performance Optimizations
- **Smart caching:** Cargo, pip, and sccache caching strategies
- **Parallel execution:** Matrix strategy for workspace members
- **Conditional jobs:** Skip irrelevant language checks automatically
- **Configurable timeouts:** Prevent hung CI runs

## Repository Structure

```
Coldaine/.github/
├── .github/
│   └── workflows/
│       ├── lang-ci.yml                    # Router workflow (triggers on push/PR)
│       └── reusable/
│           ├── python-ci.yml              # Reusable Python CI workflow
│           └── rust-ci.yml                # Reusable Rust CI workflow
├── Reusable_Workflow_Design.md           # This design document
└── README.md                              # Repository documentation
```

## Workflow Jobs

### Router Workflow (`lang-ci.yml`)

#### 1. **Path-based Detection** (`detect`)
- Uses `dorny/paths-filter` for efficient file change detection
- Checks for Python and Rust related files
- Outputs boolean flags for language presence
- Enables smart-skipping of irrelevant CI jobs

#### 2. **Python CI** (`python`)
- Conditionally calls `reusable/python-ci.yml` if Python files detected
- Passes configurable inputs for Python-specific settings
- Includes per-job concurrency control

#### 3. **Rust CI** (`rust`)  
- Conditionally calls `reusable/rust-ci.yml` if Rust files detected
- Passes extensive Rust configuration options
- Includes per-job concurrency control

### Reusable Python Workflow (`reusable/python-ci.yml`)

#### 1. **Python CI** (`py`)
- Matrix testing across multiple Python versions
- Auto-detects package managers (uv, poetry, pdm, pip)
- Runs linting, type checking, testing, coverage, and security checks
- Configurable steps with input toggles

### Reusable Rust Workflow (`reusable/rust-ci.yml`)

#### 1. **Workspace Member Detection** (`detect-members`)
- Discovers Rust workspace members for parallel testing
- Creates matrix for per-crate jobs

#### 2. **Rust CI** (`rust`)
- Per-crate parallel testing with matrix strategy
- Full CI pipeline: format, clippy, build, test, coverage
- Configurable toolchain, features, and dependencies

#### 3. **Rust MSRV** (`rust-msrv`)
- Optional MSRV testing with minimal features

#### 4. **Cargo Deny** (`cargo-deny`)
- License compliance and security checks

## Configuration Options

The workflow accepts 16 configurable inputs grouped by category:

### Language Selection
- `run_rust` (bool): Enable Rust CI jobs
- `run_python` (bool): Enable Python CI jobs

### Rust Configuration  
- `rust_toolchain`: Toolchain version (stable/beta/nightly)
- `rust_msrv`: Minimum supported Rust version
- `rust_features`: Space-separated feature flags
- `rust_no_default_features`: Disable default features
- `rust_workspace_members`: Specific members to test

### Python Configuration
- `python_version`: Python version (default: 3.11)
- `python_requirements`: Path to requirements file

### System Dependencies
- `install_alsa`: Install ALSA audio libraries
- `install_apt_packages`: Additional apt packages
- `crate_system_deps`: JSON mapping crates to system deps

### Testing Options
- `test_timeout_minutes`: Test timeout (default: 30)
- `continue_on_error`: Continue on test failures
- `use_nextest`: Use cargo-nextest for Rust
- `use_sccache`: Enable sccache caching
- `run_cargo_deny`: Run license/security checks
- `max_parallel`: Limit parallel matrix jobs

## Usage Examples

### Basic Rust Project
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  ci:
    uses: coldaine/.github/.github/workflows/lang-ci.yml@v1
    with:
      run_rust: true
      run_python: false
```

### Multi-Crate Workspace
```yaml
jobs:
  ci:
    uses: coldaine/.github/.github/workflows/lang-ci.yml@v1
    with:
      run_rust: true
      rust_workspace_members: "audio vad stt"
      crate_system_deps: '{"audio": "libasound2-dev"}'
      use_nextest: true
      use_sccache: true
```

### Python Project
```yaml
jobs:
  ci:
    uses: coldaine/.github/.github/workflows/lang-ci.yml@v1
    with:
      run_rust: false
      run_python: true
      python_version: "3.12"
      install_apt_packages: "libpq-dev"
```

### Mixed Language Project
```yaml
jobs:
  ci:
    uses: coldaine/.github/.github/workflows/lang-ci.yml@v1
    with:
      run_rust: true
      run_python: true
      rust_features: "cli"
      python_version: "3.11"
```

## Implementation Status

✅ **Complete:**
- Repository created at `Coldaine/.github`
- Workflow file implemented (`.github/workflows/lang-ci.yml`)
- All 6 specialized CI jobs configured
- 16 configurable input parameters
- Security best practices (SHA pinning)
- Performance optimizations (caching, parallel execution)
- Auto-detection capabilities

## Benefits

### For Development Teams
- **Consistency:** Same CI standards across all projects
- **Reduced Maintenance:** Single workflow to update
- **Best Practices:** Security and performance built-in
- **Flexibility:** Extensive configuration options

### For DevOps
- **Centralized Management:** One workflow for all repos
- **Version Control:** Semantic versioning with tags
- **Security Compliance:** Automated vulnerability scanning
- **Cost Optimization:** Smart caching and parallel execution

## Technical Highlights

- **550+ lines** of production-ready YAML
- **7 specialized jobs** including precheck for file detection
- **SHA-pinned actions** for supply chain security
- **Matrix strategy** for parallel workspace testing
- **Smart detection** of project type and structure
- **Comprehensive testing** including MSRV and security audits
- **Proper handling** of `hashFiles()` limitations in reusable workflows

## Future Enhancements

Potential additions for v2:
- Container image building and scanning
- Performance benchmarking integration
- Code coverage reporting to external services
- Release automation hooks
- Cross-compilation support
- GPU-accelerated testing support

## Support & Maintenance

- **Issues:** Report at [Coldaine/.github/issues](https://github.com/Coldaine/.github/issues)
- **Updates:** Managed via Dependabot (weekly)
- **Security:** Automated vulnerability scanning
- **Monitoring:** GitHub Actions insights dashboard

---

*This reusable workflow represents modern CI/CD best practices for 2025, combining security, performance, and maintainability in a single, organization-wide solution.*