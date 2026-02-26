# GitHub Actions Workflows Architecture

This document provides a comprehensive overview of TheRock's CI/CD workflow system, including workflow relationships, trigger mechanisms, and the Python scripts that configure test matrices.

## Table of Contents

- [Workflow Overview](#workflow-overview)
- [Entry Point Workflows](#entry-point-workflows)
- [Reusable Workflows](#reusable-workflows)
- [Workflow Call Hierarchy](#workflow-call-hierarchy)
- [AMDGPU Matrix Configuration](#amdgpu-matrix-configuration)
- [CI Configuration Scripts](#ci-configuration-scripts)
- [Test Configuration System](#test-configuration-system)
- [Event-to-Test Coverage Mapping](#event-to-test-coverage-mapping)
- [Runner Configuration](#runner-configuration)

## Workflow Overview

TheRock uses a hierarchical workflow system where entry point workflows dispatch to reusable workflows. This design enables:

- Code reuse across CI, nightly, and release pipelines
- Parallel execution of independent build/test jobs
- Flexible matrix expansion based on GPU families and platforms

### Workflow Categories

| Category | Purpose | Examples |
|----------|---------|----------|
| CI Entry Points | Triggered by push/PR/schedule | `ci.yml`, `ci_nightly.yml`, `ci_asan.yml` |
| Platform CI | Platform-specific build/test orchestration | `ci_linux.yml`, `ci_windows.yml` |
| Build Artifacts | Compile ROCm components | `build_portable_linux_artifacts.yml` |
| Test Execution | Run tests against built artifacts | `test_artifacts.yml`, `test_component.yml` |
| Python Packages | Build Python wheels | `build_portable_linux_python_packages.yml` |
| PyTorch Integration | Build/test PyTorch wheels | `build_portable_linux_pytorch_wheels.yml` |
| Release | Publish packages and wheels | `release_portable_linux_packages.yml` |
| Docker | Build and publish container images | `publish_dockerfile.yml` |
| Multi-Arch | Cross-architecture builds | `multi_arch_ci.yml`, `multi_arch_build_portable_linux.yml` |

## Entry Point Workflows

These workflows can be triggered directly via `workflow_dispatch`, push, pull request, or schedule:

### Primary CI Workflows

| Workflow | Triggers | Description |
|----------|----------|-------------|
| `ci.yml` | push (main, release/*), pull_request, workflow_dispatch | Main CI entry point for Linux and Windows |
| `ci_nightly.yml` | schedule (2 AM UTC), workflow_dispatch | Nightly CI with all GPU families |
| `ci_asan.yml` | schedule (2 AM UTC), workflow_dispatch | Address Sanitizer builds |
| `ci_tsan.yml` | workflow_dispatch | Thread Sanitizer builds |
| `ci_weekly.yml` | workflow_dispatch | Weekly CI placeholder |
| `multi_arch_ci.yml` | push (main, multi_arch/**), workflow_dispatch | Multi-architecture CI builds |

### Release Workflows

| Workflow | Triggers | Description |
|----------|----------|-------------|
| `release_portable_linux_packages.yml` | workflow_call, workflow_dispatch, schedule (4 AM UTC) | Linux package releases |
| `release_windows_packages.yml` | workflow_call, workflow_dispatch, schedule (4 AM UTC) | Windows package releases |

### Testing Workflows (Direct Dispatch)

| Workflow | Triggers | Description |
|----------|----------|-------------|
| `test_artifacts.yml` | workflow_dispatch, workflow_call, push (ADHOCBUILD) | Test built artifacts |
| `test_benchmarks.yml` | workflow_dispatch, workflow_call | Run performance benchmarks |
| `test_sanity_check.yml` | workflow_dispatch, workflow_call, push (ADHOCBUILD) | Basic sanity checks |
| `therock_test_harness.yml` | workflow_dispatch | Manual test harness execution |
| `unit_tests.yml` | push (main, release/*), pull_request, workflow_dispatch | Python unit tests |

### Docker/Image Workflows

| Workflow | Triggers | Description |
|----------|----------|-------------|
| `publish_build_manylinux_x86_64.yml` | workflow_dispatch, push (main, stage/docker/**) | Build manylinux image |
| `publish_build_manylinux_rccl_x86_64.yml` | workflow_dispatch, push (main, stage/docker/**) | Build manylinux RCCL image |
| `publish_no_rocm_image_ubuntu24_04.yml` | workflow_dispatch, push (main, stage/docker/**) | Ubuntu 24.04 base image |

### Other Workflows

| Workflow | Triggers | Description |
|----------|----------|-------------|
| `pre-commit.yml` | pull_request, push (main) | Code formatting checks |
| `hip_tagging_automation.yml` | push (main) | HIP version tagging |
| `copy_release.yml` | workflow_dispatch | Copy releases between locations |

## Reusable Workflows

These workflows are called by other workflows via `workflow_call`:

### Setup and Configuration

| Workflow | Purpose | Key Inputs |
|----------|---------|------------|
| `setup.yml` | Generate CI matrix configuration | `build_variant`, `multi_arch` |

### Build Workflows

| Workflow | Platform | Purpose |
|----------|----------|---------|
| `build_portable_linux_artifacts.yml` | Linux | Build ROCm artifacts |
| `build_windows_artifacts.yml` | Windows | Build ROCm artifacts |
| `build_portable_linux_python_packages.yml` | Linux | Build Python packages |
| `build_windows_python_packages.yml` | Windows | Build Python packages |
| `build_portable_linux_pytorch_wheels.yml` | Linux | Build PyTorch wheels |
| `build_windows_pytorch_wheels.yml` | Windows | Build PyTorch wheels |
| `build_portable_linux_pytorch_wheels_ci.yml` | Linux | CI PyTorch wheel builds |
| `build_windows_pytorch_wheels_ci.yml` | Windows | CI PyTorch wheel builds |
| `build_linux_jax_wheels.yml` | Linux | Build JAX wheels |
| `build_native_linux_packages.yml` | Linux | Build native packages (deb/rpm) |

### Multi-Architecture Build Workflows

| Workflow | Platform | Purpose |
|----------|----------|---------|
| `multi_arch_build_portable_linux.yml` | Linux | Orchestrate multi-arch builds |
| `multi_arch_build_windows.yml` | Windows | Orchestrate multi-arch builds |
| `multi_arch_build_portable_linux_artifacts.yml` | Linux | Per-stage artifact builds |
| `multi_arch_build_windows_artifacts.yml` | Windows | Per-stage artifact builds |
| `multi_arch_ci_linux.yml` | Linux | Multi-arch CI orchestration |
| `multi_arch_ci_windows.yml` | Windows | Multi-arch CI orchestration |

### Test Workflows

| Workflow | Purpose |
|----------|---------|
| `test_component.yml` | Run component-specific tests |
| `test_pytorch_wheels.yml` | Test PyTorch wheel functionality |
| `test_linux_jax_wheels.yml` | Test JAX wheel functionality |
| `test_rocm_wheels.yml` | Test ROCm Python wheels |
| `test_jax_dockerfile.yml` | Test JAX Docker image |

### Release Workflows

| Workflow | Purpose |
|----------|---------|
| `release_native_linux_packages.yml` | Release native packages |
| `release_portable_linux_pytorch_wheels.yml` | Release Linux PyTorch wheels |
| `release_windows_pytorch_wheels.yml` | Release Windows PyTorch wheels |
| `release_portable_linux_jax_wheels.yml` | Release JAX wheels |

### Utility Workflows

| Workflow | Purpose |
|----------|---------|
| `publish_dockerfile.yml` | Generic Dockerfile publishing |

## Workflow Call Hierarchy

### Main CI Flow (ci.yml)

```
ci.yml
├── setup.yml (generate matrix)
├── ci_linux.yml
│   ├── build_portable_linux_artifacts.yml
│   ├── test_artifacts.yml
│   │   ├── test_sanity_check.yml
│   │   └── test_component.yml (multiple shards)
│   ├── test_benchmarks.yml
│   │   └── test_component.yml
│   ├── build_portable_linux_python_packages.yml
│   ├── test_rocm_wheels.yml
│   └── build_portable_linux_pytorch_wheels_ci.yml
└── ci_windows.yml
    ├── build_windows_artifacts.yml
    ├── test_artifacts.yml
    │   ├── test_sanity_check.yml
    │   └── test_component.yml (multiple shards)
    ├── test_benchmarks.yml
    │   └── test_component.yml
    ├── build_windows_python_packages.yml
    ├── test_rocm_wheels.yml
    └── build_windows_pytorch_wheels_ci.yml
```

### Multi-Architecture CI Flow (multi_arch_ci.yml)

```
multi_arch_ci.yml
├── setup.yml (multi_arch=true)
├── multi_arch_ci_linux.yml
│   ├── multi_arch_build_portable_linux.yml
│   │   └── multi_arch_build_portable_linux_artifacts.yml (per-stage)
│   │       ├── foundation
│   │       ├── compiler-runtime
│   │       ├── math-libs
│   │       ├── comm-libs
│   │       ├── debug-tools
│   │       ├── dctools-core
│   │       ├── profiler-apps
│   │       └── media-libs
│   └── test_artifacts.yml (per-family)
└── multi_arch_ci_windows.yml
    ├── multi_arch_build_windows.yml
    │   └── multi_arch_build_windows_artifacts.yml (per-stage)
    └── test_artifacts.yml (per-family)
```

### Release Flow (release_portable_linux_packages.yml)

```
release_portable_linux_packages.yml
├── Build stages (foundation → math-libs → comm-libs → ...)
├── Merge all artifacts
├── Test artifacts (test_artifacts.yml)
├── Upload to S3
├── build_native_linux_packages.yml (deb)
├── build_native_linux_packages.yml (rpm)
├── release_portable_linux_jax_wheels.yml
│   └── build_linux_jax_wheels.yml (matrix)
└── release_portable_linux_pytorch_wheels.yml
    └── build_portable_linux_pytorch_wheels.yml (matrix)
        └── test_pytorch_wheels.yml
```

## AMDGPU Matrix Configuration

The AMDGPU matrix is defined in [`build_tools/github_actions/amdgpu_family_matrix.py`](../../build_tools/github_actions/amdgpu_family_matrix.py) and controls which GPU families are built and tested.

### Matrix Categories

| Category | When it Runs | GPU Families |
|----------|--------------|--------------|
| **Presubmit** | All PRs and pushes | gfx94x (MI300), gfx110x, gfx1151, gfx120x |
| **Postsubmit** | Main branch pushes | gfx950 (MI355/MI370) |
| **Nightly** | Scheduled runs only | gfx906, gfx908, gfx90a, gfx101x, gfx103x, gfx1150, gfx1152, gfx1153 |

### Family Configuration Structure

Each GPU family entry contains:

```python
{
    "family": "gfx94X-dcgpu",                    # CMake target family
    "test-runs-on": "linux-mi325-1gpu-ossci-rocm",  # Test runner label
    "test-runs-on-sandbox": "...",               # Sandbox runner (ASAN)
    "test-runs-on-multi-gpu": "...",             # Multi-GPU runner
    "benchmark-runs-on": "...",                  # Benchmark runner
    "fetch-gfx-targets": ["gfx942"],             # Individual GPU targets
    "build_variants": ["release", "asan", "tsan"],  # Available variants
    "bypass_tests_for_releases": False,          # Skip tests on release
    "sanity_check_only_for_family": False,       # Only sanity checks
    "run-full-tests-only": False,                # Skip smoke tests
}
```

### Build Variants

| Variant | Platform | Description |
|---------|----------|-------------|
| `release` | Linux, Windows | Standard release build |
| `asan` | Linux only | Address Sanitizer build |
| `tsan` | Linux only | Thread Sanitizer build |

## CI Configuration Scripts

### configure_ci.py

**Location:** [`build_tools/github_actions/configure_ci.py`](../../build_tools/github_actions/configure_ci.py)

**Purpose:** Generates the CI matrix based on GitHub event type.

**Key Inputs:**
- `GITHUB_EVENT_NAME` - Trigger type (push, pull_request, schedule, workflow_dispatch)
- `GITHUB_REF_NAME` - Branch name
- `PR_LABELS` - Labels on the PR
- `BUILD_VARIANT` - Build variant (release/asan/tsan)
- `MULTI_ARCH` - Enable multi-architecture mode

**Matrix Generation by Event:**

| Event | Families Used | Test Type |
|-------|--------------|-----------|
| `pull_request` | Presubmit only | Smoke (unless labels specify otherwise) |
| `push` (main) | Presubmit + Postsubmit | Full or smoke based on changes |
| `push` (other) | Presubmit only | Smoke |
| `schedule` | All (presubmit + postsubmit + nightly) | Full |
| `workflow_dispatch` | User-specified | User-specified |

### configure_ci_path_filters.py

**Location:** [`build_tools/github_actions/configure_ci_path_filters.py`](../../build_tools/github_actions/configure_ci_path_filters.py)

**Purpose:** Determines if CI should run based on changed files.

**Skipped Paths (CI does not run):**
- `docs/*` - Documentation
- `*.md` - Markdown files
- `.pre-commit-config.*` - Pre-commit configuration
- `.github/dependabot.yml` - Dependabot config
- `dockerfiles/*` - Docker images (separate CI)
- `experimental/*` - Experimental code

**CI-Triggering Workflow Files:**
Changes to these files always trigger CI:
- `setup.yml`, `ci*.yml`, `multi_arch*.yml`
- `build*artifact*.yml`, `build*ci.yml`, `build*python_packages.yml`
- `test*artifacts.yml`, `test_rocm_wheels.yml`, `test_sanity_check.yml`, `test_component.yml`

### configure_target_run.py

**Location:** [`build_tools/github_actions/configure_target_run.py`](../../build_tools/github_actions/configure_target_run.py)

**Purpose:** Resolves GPU family to actual test runner label.

**Functions:**
- `get_runner_label(target, platform)` - Maps GPU family to runner
- `get_upload_label(target, platform)` - Gets bypass_tests_for_releases flag

## Test Configuration System

### fetch_test_configurations.py

**Location:** [`build_tools/github_actions/fetch_test_configurations.py`](../../build_tools/github_actions/fetch_test_configurations.py)

**Purpose:** Determines which tests run and their configuration.

**Test Entry Structure:**

```python
{
    "job_name": "rocblas",
    "fetch_artifact_args": "--blas --tests",
    "timeout_minutes": 15,
    "test_script": "python build_tools/github_actions/test_executable_scripts/test_rocblas.py",
    "platform": ["linux", "windows"],
    "total_shards": 1,
    "exclude_family": {
        "linux": ["gfx1150", "gfx1151"],
        "windows": ["gfx1151"]
    },
    "multi_gpu": {
        "linux": ["gfx94X-dcgpu"]
    }
}
```

### Test Categories

| Category | Tests |
|----------|-------|
| BLAS/Linear Algebra | rocblas, rocroller, hipblas, hipblaslt, rocsolver, hipsolver |
| Sparse | rocsparse, hipsparse, hipsparselt |
| Primitives | rocprim, hipcub, rocthrust |
| FFT | rocfft, hipfft |
| Random | rocrand, hiprand |
| ML/DNN | miopen, hipdnn, miopenprovider, hipblasltprovider |
| Communication | rccl |
| System/Tools | rocprofiler_systems, rocrtst, aqlprofile, rocr-debug-agent |
| Core | hip-tests |
| Hardware | rocwmma, libhipcxx_hipcc, libhipcxx_hiprtc |

### Test Types

| Type | Description | When Used |
|------|-------------|-----------|
| **Smoke** | Single shard, quick validation | PRs, pushes without submodule changes |
| **Full** | All shards, comprehensive | Nightly, submodule changes, test labels |

### Benchmarks

Benchmarks run separately from regular tests (nightly only):

| Benchmark | Platform |
|-----------|----------|
| rocblas_bench | Linux |
| hipblaslt_bench | Linux, Windows |
| rocsolver_bench | Linux, Windows |
| rocrand_bench | Linux, Windows |
| rocfft_bench | Linux, Windows |

## Event-to-Test Coverage Mapping

### Pull Request

```
Trigger: pull_request
Matrix: Presubmit families only
GPU Families: gfx94x, gfx110x, gfx1151, gfx120x
Build Variants: release
Test Type: Smoke
Optional: PR labels can opt-in to additional families/tests
```

**PR Label Options:**
- `skip-ci` - Skip all CI
- `run-all-archs-ci` - Enable all architectures
- `gfx...` - Add specific GPU family (e.g., `gfx950`)
- `test:...` - Run full tests for specific component (e.g., `test:rocblas`)
- `test_runner:oem` - Use OEM kernel test runners

### Push to Main

```
Trigger: push (main branch)
Matrix: Presubmit + Postsubmit
GPU Families: gfx94x, gfx110x, gfx1151, gfx120x, gfx950
Build Variants: release (+ asan/tsan if configured)
Test Type: Full or smoke based on path changes
```

### Scheduled (Nightly)

```
Trigger: schedule (2 AM UTC)
Matrix: All (Presubmit + Postsubmit + Nightly)
GPU Families: All available
Build Variants: release + configured variants
Test Type: Always full
Benchmarks: Included
```

### Workflow Dispatch

```
Trigger: workflow_dispatch
Matrix: User-specified families
GPU Families: Any available
Test Labels: User-specified
Build Variant: User-specified
Test Type: Full if test labels specified
```

## Runner Configuration

### Linux Runners

| Runner Label | GPU | Purpose |
|--------------|-----|---------|
| `linux-mi325-1gpu-ossci-rocm` | MI325 (gfx942) | Single GPU tests |
| `linux-mi325-8gpu-ossci-rocm` | 8× MI325 | Multi-GPU tests, benchmarks |
| `linux-mi325-8gpu-ossci-rocm-sandbox` | 8× MI325 | ASAN sandbox testing |
| `linux-mi355-1gpu-ossci-rocm` | MI355 (gfx950) | Next-gen GPU tests |
| `linux-gfx1151-gpu-rocm` | Radeon PRO (gfx1151) | Desktop GPU tests |
| `linux-gfx110X-gpu-rocm` | Instinct (gfx110x) | Instinct tests |
| `linux-gfx120X-gpu-rocm` | MI310/320 (gfx120x) | MI310/320 tests |
| `linux-strix-halo-gpu-rocm-oem` | Strix Halo | OEM kernel testing |

### Windows Runners

| Runner Label | GPU | Purpose |
|--------------|-----|---------|
| `windows-gfx110X-gpu-rocm` | Instinct (gfx110x) | Instinct tests |
| `windows-gfx1151-gpu-rocm` | Radeon PRO (gfx1151) | Desktop GPU tests |
| `windows-gfx120X-gpu-rocm` | MI310/320 (gfx120x) | MI310/320 tests |

### Runner Selection Logic

1. `configure_ci.py` generates matrix with GPU families
2. `configure_target_run.py` resolves family to runner label
3. Tests with `multi_gpu` requirement get multi-GPU runner
4. ASAN builds use sandbox runners
5. Benchmarks use dedicated benchmark runners

## System Interdependencies

```
amdgpu_family_matrix.py
  ↓ (GPU families, runners, variants)
configure_ci.py
  ↓ (generates matrix based on event)
configure_target_run.py
  ↓ (resolves runner labels)
fetch_test_configurations.py
  ↓ (selects tests, applies sharding)
GitHub Workflow
  ↓ (executes jobs)
Test Results
```

## Related Documentation

- [CI Behavior Manipulation](ci_behavior_manipulation.md) - PR labels and event-based behavior
- [GitHub Actions Debugging](github_actions_debugging.md) - Debugging CI failures
- [Test Filtering](test_filtering.md) - Component test filtering
- [Test Runner Info](test_runner_info.md) - Runner specifications
- [Adding Tests](adding_tests.md) - How to add new tests
