# Workflow Call Chains

This document maps out the entry point workflows and their complete call chains, showing which reusable workflows and scripts each workflow invokes.

## Entry Point Workflows

Entry point workflows can be triggered directly via push, pull request, schedule, or manual dispatch. They orchestrate the CI/CD pipeline by calling reusable workflows.

| Entry Workflow | Triggers | Purpose |
|---------------|----------|---------|
| [`ci.yml`](#ci-workflow) | push, PR, dispatch | Main CI for PRs and main branch |
| [`ci_nightly.yml`](#ci-nightly-workflow) | schedule (2 AM UTC), dispatch | All GPU families + benchmarks |
| [`ci_asan.yml`](#ci-asan-workflow) | schedule (2 AM UTC), dispatch | Linux ASAN builds |
| [`ci_tsan.yml`](#ci-tsan-workflow) | dispatch | Linux TSAN builds |
| [`multi_arch_ci.yml`](#multi-arch-ci-workflow) | push, dispatch | Staged multi-architecture builds |
| [`release_portable_linux_packages.yml`](#release-linux-workflow) | schedule (4 AM UTC), dispatch, workflow_call | Linux release packages |
| [`release_windows_packages.yml`](#release-windows-workflow) | schedule (4 AM UTC), dispatch, workflow_call | Windows release packages |

---

## CI Workflow

> **File:** [`.github/workflows/ci.yml`](../../.github/workflows/ci.yml)

**Triggers:**
- Push to `main` or `release/therock-*` branches
- Pull requests (labeled, opened, synchronized)
- Manual dispatch

### Call Chain

```
ci.yml
│
├─► setup.yml
│   │
│   └─► Scripts:
│       ├── configure_ci.py              # Determines GPU families and variants
│       └── compute_rocm_package_version.py  # Computes package version
│
├─► ci_linux.yml (matrix: per GPU family)
│   │
│   ├─► build_portable_linux_artifacts.yml
│   │   │
│   │   └─► Scripts:
│   │       ├── build_configure.py       # CMake configuration
│   │       ├── resource_info.py         # Resource usage tracking
│   │       ├── analyze_build_times.py   # Build timing analysis
│   │       └── post_build_upload.py     # S3 artifact upload
│   │
│   ├─► test_artifacts.yml
│   │   │
│   │   ├─► Scripts:
│   │   │   └── fetch_test_configurations.py  # Test matrix generation
│   │   │
│   │   ├─► test_sanity_check.yml
│   │   │
│   │   └─► test_component.yml (matrix: per test)
│   │       │
│   │       └─► Scripts:
│   │           ├── fetch_artifacts.py   # Download artifacts from S3
│   │           └── test_*.py            # Component test scripts
│   │
│   ├─► test_benchmarks.yml
│   │   │
│   │   └─► test_component.yml (benchmark tests)
│   │
│   ├─► build_portable_linux_python_packages.yml
│   │   │
│   │   └─► Scripts:
│   │       └── build_python_packages.py
│   │
│   ├─► test_rocm_wheels.yml
│   │
│   └─► build_portable_linux_pytorch_wheels_ci.yml
│
└─► ci_windows.yml (matrix: per GPU family)
    │
    ├─► build_windows_artifacts.yml
    │
    ├─► test_artifacts.yml
    │   ├─► test_sanity_check.yml
    │   └─► test_component.yml
    │
    ├─► test_benchmarks.yml
    │
    ├─► build_windows_python_packages.yml
    │
    ├─► test_rocm_wheels.yml
    │
    └─► build_windows_pytorch_wheels_ci.yml
```

---

## CI Nightly Workflow

> **File:** [`.github/workflows/ci_nightly.yml`](../../.github/workflows/ci_nightly.yml)

**Triggers:**
- Schedule: Daily at 2 AM UTC
- Manual dispatch

### Call Chain

```
ci_nightly.yml
│
├─► setup.yml
│   └─► Scripts:
│       ├── configure_ci.py              # Selects ALL GPU families for nightly
│       └── compute_rocm_package_version.py
│
├─► ci_linux.yml (matrix: ALL GPU families)
│   │
│   │   Includes benchmark_runs_on parameter for performance testing
│   │
│   ├─► build_portable_linux_artifacts.yml
│   ├─► test_artifacts.yml
│   │   ├─► test_sanity_check.yml
│   │   └─► test_component.yml (full shards)
│   ├─► test_benchmarks.yml              # Runs nightly benchmarks
│   │   └─► test_component.yml (benchmark_matrix)
│   ├─► build_portable_linux_python_packages.yml
│   ├─► test_rocm_wheels.yml
│   └─► build_portable_linux_pytorch_wheels_ci.yml
│
└─► ci_windows.yml (matrix: ALL GPU families)
    └─► (same structure as Linux)
```

**Key Difference from ci.yml:** Runs ALL GPU families (presubmit + postsubmit + nightly) and includes benchmark tests.

---

## CI ASAN Workflow

> **File:** [`.github/workflows/ci_asan.yml`](../../.github/workflows/ci_asan.yml)

**Triggers:**
- Schedule: Daily at 2 AM UTC
- Manual dispatch

### Call Chain

```
ci_asan.yml
│
├─► setup.yml (build_variant: "asan")
│   └─► Scripts:
│       └── configure_ci.py              # Filters for ASAN-compatible families
│
└─► ci_linux.yml (ASAN variant only, no Windows)
    │
    ├─► build_portable_linux_artifacts.yml
    │   │
    │   │   Uses: azure-linux-scale-rocm-heavy runner (more resources)
    │   │   CMake preset: linux-release-asan
    │   │
    │   └─► Scripts:
    │       └── build_configure.py
    │
    ├─► test_artifacts.yml
    │   └─► test_component.yml (ASAN instrumented tests)
    │
    └─► build_portable_linux_python_packages.yml
```

**Key Differences:**
- Linux only (no Windows ASAN support)
- Uses heavier runners for resource-intensive builds
- No benchmarks or PyTorch wheels

---

## CI TSAN Workflow

> **File:** [`.github/workflows/ci_tsan.yml`](../../.github/workflows/ci_tsan.yml)

**Triggers:**
- Manual dispatch only

### Call Chain

```
ci_tsan.yml
│
├─► setup.yml (build_variant: "tsan")
│
└─► ci_linux.yml (TSAN variant only)
    │
    ├─► build_portable_linux_artifacts.yml
    │   │
    │   │   CMake preset: linux-release-tsan
    │   │
    │   └─► Scripts:
    │       └── build_configure.py
    │
    └─► test_artifacts.yml
```

---

## Multi-Arch CI Workflow

> **File:** [`.github/workflows/multi_arch_ci.yml`](../../.github/workflows/multi_arch_ci.yml)

**Triggers:**
- Push to `main` or `multi_arch/**` branches
- Manual dispatch

### Call Chain

```
multi_arch_ci.yml
│
├─► setup.yml (multi_arch: true)
│   └─► Scripts:
│       └── configure_ci.py              # Groups families into single entry
│
├─► multi_arch_ci_linux.yml
│   │
│   ├─► multi_arch_build_portable_linux.yml
│   │   │
│   │   │   Staged build pipeline - each stage depends on previous:
│   │   │
│   │   ├─► multi_arch_build_portable_linux_artifacts.yml (stage: foundation)
│   │   │   └─► Scripts:
│   │   │       └── artifact_manager.py  # S3 artifact flow between stages
│   │   │
│   │   ├─► multi_arch_build_portable_linux_artifacts.yml (stage: compiler-runtime)
│   │   │
│   │   ├─► multi_arch_build_portable_linux_artifacts.yml (stage: math-libs)
│   │   │   │
│   │   │   └─► Matrix: per GPU architecture
│   │   │
│   │   ├─► multi_arch_build_portable_linux_artifacts.yml (stage: comm-libs)
│   │   │
│   │   ├─► multi_arch_build_portable_linux_artifacts.yml (stage: debug-tools)
│   │   │
│   │   ├─► multi_arch_build_portable_linux_artifacts.yml (stage: dctools-core)
│   │   │
│   │   ├─► multi_arch_build_portable_linux_artifacts.yml (stage: profiler-apps)
│   │   │
│   │   └─► multi_arch_build_portable_linux_artifacts.yml (stage: media-libs)
│   │
│   └─► test_artifacts.yml (matrix: per GPU family)
│
└─► multi_arch_ci_windows.yml
    │
    ├─► multi_arch_build_windows.yml
    │   ├─► multi_arch_build_windows_artifacts.yml (stage: foundation)
    │   ├─► multi_arch_build_windows_artifacts.yml (stage: compiler-runtime)
    │   └─► multi_arch_build_windows_artifacts.yml (stage: math-libs)
    │
    └─► test_artifacts.yml
```

**Key Feature:** Sharded multi-stage build pipeline with S3-based artifact flow between stages.

---

## Release Linux Workflow

> **File:** [`.github/workflows/release_portable_linux_packages.yml`](../../.github/workflows/release_portable_linux_packages.yml)

**Triggers:**
- Schedule: Daily at 4 AM UTC
- Manual dispatch
- Workflow call from other workflows

### Call Chain

```
release_portable_linux_packages.yml
│
├─► Job: setup_metadata
│   └─► Scripts:
│       ├── compute_rocm_package_version.py  # Computes release version
│       └── fetch_package_targets.py         # Determines GPU family matrix
│
├─► Job: portable_linux_packages (matrix: per GPU family)
│   │
│   └─► Scripts:
│       ├── build_configure.py           # CMake configuration
│       ├── cmake --build                # Build therock-archives, therock-dist
│       ├── build_python_packages.py     # Build Python wheels
│       ├── analyze_build_times.py       # Build timing analysis
│       ├── post_build_upload.py         # Upload artifacts to S3
│       ├── manage.py                    # Generate S3 pip index
│       └── index_generation_s3_tar.py   # Generate tarball index
│
└─► Triggers external workflows (via benc-uk/workflow-dispatch):
    │
    ├─► release_portable_linux_pytorch_wheels.yml
    │   └─► build_portable_linux_pytorch_wheels.yml
    │       └─► test_pytorch_wheels.yml
    │
    ├─► release_portable_linux_jax_wheels.yml
    │   └─► build_linux_jax_wheels.yml
    │       └─► test_linux_jax_wheels.yml
    │
    ├─► build_native_linux_packages.yml (RPM)
    │
    └─► build_native_linux_packages.yml (DEB)
```

**Key Feature:** Builds distributable packages and triggers downstream wheel/package builds.

---

## Release Windows Workflow

> **File:** [`.github/workflows/release_windows_packages.yml`](../../.github/workflows/release_windows_packages.yml)

**Triggers:**
- Schedule: Daily at 4 AM UTC
- Manual dispatch
- Workflow call from other workflows

### Call Chain

```
release_windows_packages.yml
│
├─► Job: setup_metadata
│   └─► Scripts:
│       ├── compute_rocm_package_version.py
│       └── fetch_package_targets.py
│
├─► Job: windows_packages (matrix: per GPU family)
│   │
│   └─► Scripts:
│       ├── build_configure.py
│       ├── cmake --build
│       ├── build_python_packages.py
│       ├── post_build_upload.py
│       ├── manage.py
│       └── index_generation_s3_tar.py
│
└─► Triggers external workflow:
    │
    └─► release_windows_pytorch_wheels.yml
        └─► build_windows_pytorch_wheels.yml
            └─► test_pytorch_wheels.yml
```

---

## Key Scripts Reference

| Script | Location | Purpose |
|--------|----------|---------|
| `configure_ci.py` | `build_tools/github_actions/` | Determines GPU families and build variants based on event |
| `compute_rocm_package_version.py` | `build_tools/` | Computes package version with appropriate suffix |
| `fetch_package_targets.py` | `build_tools/github_actions/` | Generates GPU family matrix for releases |
| `fetch_test_configurations.py` | `build_tools/github_actions/` | Generates test matrix from `test_matrix` |
| `build_configure.py` | `build_tools/github_actions/` | Configures CMake build |
| `build_python_packages.py` | `build_tools/` | Builds Python wheels from artifacts |
| `post_build_upload.py` | `build_tools/github_actions/` | Uploads artifacts and logs to S3 |
| `fetch_artifacts.py` | `build_tools/github_actions/` | Downloads artifacts from S3 for testing |
| `artifact_manager.py` | `build_tools/github_actions/` | Manages S3 artifact flow for multi-arch builds |
| `resource_info.py` | `build_tools/` | Tracks resource usage during builds |
| `analyze_build_times.py` | `build_tools/` | Analyzes ninja build logs for timing |
| `manage.py` | `build_tools/third_party/s3_management/` | Generates pip-compatible index on S3 |
| `index_generation_s3_tar.py` | `build_tools/` | Generates tarball index on S3 |

---

## Reusable Workflows Reference

| Workflow | Purpose | Called By |
|----------|---------|-----------|
| `setup.yml` | Generate CI matrix configuration | ci.yml, ci_nightly.yml, ci_asan.yml, multi_arch_ci.yml |
| `ci_linux.yml` | Linux build/test orchestration | ci.yml, ci_nightly.yml, ci_asan.yml |
| `ci_windows.yml` | Windows build/test orchestration | ci.yml, ci_nightly.yml |
| `build_portable_linux_artifacts.yml` | Build Linux ROCm artifacts | ci_linux.yml |
| `build_windows_artifacts.yml` | Build Windows ROCm artifacts | ci_windows.yml |
| `test_artifacts.yml` | Test artifact orchestration | ci_linux.yml, ci_windows.yml, multi_arch_ci_*.yml |
| `test_sanity_check.yml` | Basic sanity checks | test_artifacts.yml |
| `test_component.yml` | Component-specific tests | test_artifacts.yml, test_benchmarks.yml |
| `test_benchmarks.yml` | Performance benchmarks | ci_linux.yml, ci_windows.yml |
| `build_portable_linux_python_packages.yml` | Build Linux Python wheels | ci_linux.yml |
| `build_windows_python_packages.yml` | Build Windows Python wheels | ci_windows.yml |
| `test_rocm_wheels.yml` | Test Python wheels | ci_linux.yml, ci_windows.yml |
| `build_portable_linux_pytorch_wheels_ci.yml` | Build PyTorch wheels (CI) | ci_linux.yml |
| `build_windows_pytorch_wheels_ci.yml` | Build PyTorch wheels (CI) | ci_windows.yml |
| `multi_arch_ci_linux.yml` | Multi-arch Linux orchestration | multi_arch_ci.yml |
| `multi_arch_ci_windows.yml` | Multi-arch Windows orchestration | multi_arch_ci.yml |
| `multi_arch_build_portable_linux.yml` | Multi-arch Linux build stages | multi_arch_ci_linux.yml |
| `multi_arch_build_portable_linux_artifacts.yml` | Per-stage Linux build | multi_arch_build_portable_linux.yml |

---

## Related Documentation

- [Workflows Architecture](workflows_architecture.md) - End-to-end CI Nightly walkthrough
- [Release and Nightly Builds](release_and_nightly_builds.md) - Package versioning and S3 structure
- [CI Behavior Manipulation](ci_behavior_manipulation.md) - PR labels and CI controls
