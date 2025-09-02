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
│       └── lang-ci.yml    # Main reusable workflow (521 lines)
└── README.md              # Repository documentation
```

## Workflow Jobs

### 1. **File Presence Detection** (`precheck`)
- Runs first to detect project type on a runner
- Checks for Rust (Cargo.toml) and Python files
- Outputs flags used by downstream jobs
- Avoids `hashFiles()` limitations in reusable workflows

### 2. **Workspace Member Detection** (`detect-members`)
- Automatically discovers Rust workspace members
- Creates a matrix for parallel testing
- Supports manual member specification
- Depends on precheck for file presence

### 3. **Rust CI** (`rust`)
- Per-crate parallel testing with matrix strategy
- Full CI pipeline: format, clippy, build, test, doc generation
- Configurable feature testing and MSRV checks
- System dependency management per crate

### 4. **Python CI** (`python`)
- Modern Python tooling with ruff and pytest
- Type checking with mypy
- Security scanning with bandit
- Coverage reporting

### 5. **Security & Dependencies**
- **cargo-deny:** License compliance and vulnerability checks
- **dependency-analysis:** Duplicate detection and security audits  
- **dependency-check:** Trivy scanning for both languages

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