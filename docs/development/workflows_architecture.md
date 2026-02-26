# GitHub Actions Workflows Architecture

This document explains how TheRock's CI/CD system works, using the CI Nightly workflow as a detailed example. For a quick reference of all workflow call chains, see [Workflow Call Chains](workflow_call_chains.md).

## Table of Contents

- [End-to-End Example: CI Nightly](#end-to-end-example-ci-nightly)
- [Understanding Event Types](#understanding-event-types)
- [How the GPU Matrix Is Created](#how-the-gpu-matrix-is-created)
- [How the Test Matrix Is Created](#how-the-test-matrix-is-created)
- [How the Benchmark Matrix Works](#how-the-benchmark-matrix-works)

## End-to-End Example: CI Nightly

This section walks through what happens when the **CI Nightly** workflow runs, from trigger to completion.

### Key Files Referenced

| File | Purpose |
|------|---------|
| [`.github/workflows/ci_nightly.yml`](../../.github/workflows/ci_nightly.yml) | Entry point workflow |
| [`.github/workflows/setup.yml`](../../.github/workflows/setup.yml) | Matrix generation workflow |
| [`.github/workflows/ci_linux.yml`](../../.github/workflows/ci_linux.yml) | Linux build/test orchestration |
| [`build_tools/github_actions/amdgpu_family_matrix.py`](../../build_tools/github_actions/amdgpu_family_matrix.py) | GPU family definitions |
| [`build_tools/github_actions/configure_ci.py`](../../build_tools/github_actions/configure_ci.py) | Selects GPU families based on trigger |
| [`build_tools/github_actions/fetch_test_configurations.py`](../../build_tools/github_actions/fetch_test_configurations.py) | Test definitions |
| [`tests/extended_tests/benchmark/benchmark_test_matrix.py`](../../tests/extended_tests/benchmark/benchmark_test_matrix.py) | Benchmark definitions |

### What Triggers It

The nightly CI runs automatically at **2 AM UTC every day** via a cron schedule. You can also trigger it manually from the GitHub Actions UI using `workflow_dispatch`.

### Step-by-Step Flow

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              ci_nightly.yml                                  │
│                         (Entry Point Workflow)                               │
└─────────────────────────────────┬────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 1: SETUP (setup.yml)                                                   │
│  ─────────────────────────                                                   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  configure_ci.py                                                    │     │
│  │  Reads: amdgpu_family_matrix.py                                     │     │
│  │                                                                     │     │
│  │  For nightly (schedule event):                                      │     │
│  │  • Selects ALL GPU families:                                        │     │
│  │    - Presubmit: gfx94x, gfx110x, gfx1151, gfx120x                   │     │
│  │    - Postsubmit: gfx950                                             │     │
│  │    - Nightly: gfx906, gfx908, gfx90a, gfx101x, gfx103x, gfx115x     │     │
│  │  • Sets test_type = "full" (all shards)                             │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                  │                                           │
│                                  ▼                                           │
│  Output: JSON matrix of GPU families for Linux and Windows                   │
│  Example: ["gfx94x", "gfx950", "gfx906", "gfx908", ...]                      │
└─────────────────────────────────┬────────────────────────────────────────────┘
                                  │
              ┌───────────────────┴───────────────────┐
              ▼                                       ▼
┌───────────────────────────┐           ┌───────────────────────────┐
│  Linux Builds             │           │  Windows Builds           │
│  (one job per GPU family) │           │  (one job per GPU family) │
│                           │           │                           │
│  gfx94x  ─┐               │           │  gfx110x ─┐               │
│  gfx950  ─┼─► parallel    │           │  gfx1151 ─┼─► parallel    │
│  gfx906  ─┤               │           │  gfx120x ─┘               │
│  gfx908  ─┤               │           │                           │
│  ...     ─┘               │           │                           │
└─────────────┬─────────────┘           └─────────────┬─────────────┘
              │                                       │
              └───────────────────┬───────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 2: PLATFORM CI (ci_linux.yml / ci_windows.yml)                         │
│  ───────────────────────────────────────────────────                         │
│  Runs for EACH GPU family in parallel. Below shows one family's flow:        │
└─────────────────────────────────┬────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 2a: BUILD ARTIFACTS (build_portable_linux_artifacts.yml)               │
│  ─────────────────────────────────────────────────────────────               │
│                                                                              │
│  • Compiles ROCm components (compiler, runtime, math libs, etc.)             │
│  • Target GPU family passed from matrix (e.g., gfx94X-dcgpu)                 │
│  • Uploads artifacts to S3 (therock-ci-artifacts bucket)                     │
└─────────────────────────────────┬────────────────────────────────────────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          ▼                       ▼                       ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│ Test Artifacts   │   │ Build Python     │   │ Benchmarks       │
│                  │   │ Packages         │   │ (nightly only)   │
└────────┬─────────┘   └────────┬─────────┘   └────────┬─────────┘
         │                      │                      │
         ▼                      ▼                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 2b: TEST ARTIFACTS (test_artifacts.yml)                                │
│  ────────────────────────────────────────────                                │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  fetch_test_configurations.py                                       │     │
│  │  Reads: test_matrix (from fetch_test_configurations.py)             │     │
│  │                                                                     │     │
│  │  Input:                                                             │     │
│  │  • AMDGPU_FAMILIES = "gfx94X-dcgpu"                                 │     │
│  │  • RUNNER_OS = "Linux"                                              │     │
│  │  • TEST_TYPE = "full"                                               │     │
│  │                                                                     │     │
│  │  Processing:                                                        │     │
│  │  • Filters tests by platform (Linux/Windows)                        │     │
│  │  • Excludes tests not supported on this GPU family                  │     │
│  │  • For full tests: uses all shards (hip-tests → 4 shards)          │     │
│  │  • For smoke tests: uses 1 shard only                               │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                  │                                           │
│                                  ▼                                           │
│  Output: JSON list of test jobs                                              │
│  Example: [                                                                  │
│    {"job_name": "hip-tests", "shard_arr": [1,2,3,4], ...},                   │
│    {"job_name": "rocblas", "shard_arr": [1], ...},                           │
│    {"job_name": "miopen", "shard_arr": [1,2,3,4], ...},                      │
│    ...                                                                       │
│  ]                                                                           │
└─────────────────────────────────┬────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 2c: TEST EXECUTION (test_component.yml)                                │
│  ────────────────────────────────────────────                                │
│                                                                              │
│  For each test in the matrix, runs in parallel:                              │
│                                                                              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐             │
│  │ hip-tests   │ │ hip-tests   │ │ hip-tests   │ │ hip-tests   │             │
│  │ shard 1/4   │ │ shard 2/4   │ │ shard 3/4   │ │ shard 4/4   │             │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐             │
│  │ rocblas     │ │ miopen 1/4  │ │ miopen 2/4  │ │ miopen 3/4  │  ...        │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘             │
│                                                                              │
│  Each test job:                                                              │
│  • Downloads artifacts for this GPU family                                   │
│  • Runs on GPU runner (from amdgpu_family_matrix.py test-runs-on)           │
│  • Executes test_script (e.g., python test_rocblas.py)                       │
│  • Reports pass/fail                                                         │
└─────────────────────────────────┬────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 2d: BENCHMARKS (test_benchmarks.yml) - Nightly Only                    │
│  ────────────────────────────────────────────────────────                    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐     │
│  │  fetch_test_configurations.py (with IS_BENCHMARK_WORKFLOW=true)     │     │
│  │  Reads: benchmark_matrix (from benchmark_test_matrix.py)            │     │
│  │                                                                     │     │
│  │  Selects benchmarks: rocblas_bench, hipblaslt_bench,                │     │
│  │                      rocsolver_bench, rocrand_bench, rocfft_bench   │     │
│  └─────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  • Runs on benchmark-runs-on runners (usually multi-GPU machines)            │
│  • No sharding (benchmarks always run as single job)                         │
│  • Measures performance metrics                                              │
└─────────────────────────────────┬────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 2e: BUILD PYTHON PACKAGES (build_portable_linux_python_packages.yml)   │
│  ─────────────────────────────────────────────────────────────────────────   │
│                                                                              │
│  • Builds Python wheels for ROCm libraries                                   │
│  • Packages versioned with computed ROCm version                             │
└─────────────────────────────────┬────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  STEP 2f: TEST PYTHON WHEELS (test_rocm_wheels.yml)                          │
│  ──────────────────────────────────────────────────                          │
│                                                                              │
│  • Installs built Python wheels                                              │
│  • Runs import tests and basic functionality checks                          │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Summary

The CI Nightly workflow:

1. **Starts** at 2 AM UTC (or manually)
2. **Selects GPU families** using `amdgpu_family_matrix.py` (all families for nightly)
3. **Builds** ROCm artifacts for each GPU family in parallel
4. **Selects tests** using `fetch_test_configurations.py` (filters by platform/family)
5. **Runs tests** on real GPU hardware with full sharding
6. **Runs benchmarks** on dedicated benchmark runners
7. **Builds and tests** Python packages

## Understanding Event Types

TheRock CI uses three event categories that determine **which GPU families are built and tested**:

| Event Type | When It Runs | GPU Families Included | Purpose |
|------------|--------------|----------------------|---------|
| **Presubmit** | Every pull request | gfx94x, gfx110x, gfx1151, gfx120x | Fast feedback for contributors |
| **Postsubmit** | Every push to `main` | Presubmit families + gfx950 | Validate merged code on more hardware |
| **Nightly** | 2 AM UTC daily | All families (presubmit + postsubmit + gfx906, gfx908, gfx90a, gfx101x, gfx103x, gfx1150, gfx1152, gfx1153) | Comprehensive coverage including older/experimental GPUs |

**Why this matters:** Not all GPU hardware is available in large quantities. Presubmit runs on the most common GPUs for quick feedback. Nightly runs on everything to catch architecture-specific issues.

## How the GPU Matrix Is Created

> **Source file:** [`build_tools/github_actions/amdgpu_family_matrix.py`](../../build_tools/github_actions/amdgpu_family_matrix.py)

This file is the **source of truth** for which GPUs exist and how to test them. It contains three dictionaries, one for each event type:

**Presubmit families** (`amdgpu_family_info_matrix_presubmit`) - run on every PR:

```python
amdgpu_family_info_matrix_presubmit = {
    "gfx94x": {
        "linux": {
            "test-runs-on": "linux-mi325-1gpu-ossci-rocm",    # Runner for tests
            "test-runs-on-sandbox": "linux-mi325-8gpu-ossci-rocm-sandbox",  # ASAN runner
            "test-runs-on-multi-gpu": "linux-mi325-8gpu-ossci-rocm",        # Multi-GPU tests
            "benchmark-runs-on": "linux-mi325-8gpu-ossci-rocm",             # Benchmarks
            "family": "gfx94X-dcgpu",                          # CMake target name
            "fetch-gfx-targets": ["gfx942"],                   # Specific GPU chips
            "build_variants": ["release", "asan", "tsan"],     # Build configurations
        }
    },
    "gfx110x": { ... },
    "gfx1151": { ... },
    "gfx120x": { ... },
}
```

**Postsubmit families** (`amdgpu_family_info_matrix_postsubmit`) - added on pushes to main:

```python
amdgpu_family_info_matrix_postsubmit = {
    "gfx950": {
        "linux": {
            "test-runs-on": "linux-mi355-1gpu-ossci-rocm",
            "family": "gfx950-dcgpu",
            "fetch-gfx-targets": ["gfx950"],
            "build_variants": ["release", "asan", "tsan"],
        }
    },
}
```

**Nightly families** (`amdgpu_family_info_matrix_nightly`) - added on scheduled runs:

```python
amdgpu_family_info_matrix_nightly = {
    "gfx906": { ... },   # Older datacenter GPU
    "gfx908": { ... },   # MI100
    "gfx90a": { ... },   # MI200 series
    "gfx101x": { ... },  # RDNA1
    "gfx103x": { ... },  # RDNA2
    "gfx1150": { ... },  # Strix Point
    "gfx1152": { ... },
    "gfx1153": { ... },
}
```

**How families are selected** ([`configure_ci.py`](../../build_tools/github_actions/configure_ci.py)):

```
pull_request  → presubmit families only
push to main  → presubmit + postsubmit families
schedule      → presubmit + postsubmit + nightly families (ALL)
```

## How the Test Matrix Is Created

> **Source file:** [`build_tools/github_actions/fetch_test_configurations.py`](../../build_tools/github_actions/fetch_test_configurations.py)

This file defines the `test_matrix` dictionary listing **what tests exist** and **how to run them**.

**Test matrix structure:**

```python
test_matrix = {
    "hip-tests": {
        "job_name": "hip-tests",
        "fetch_artifact_args": "--tests",
        "timeout_minutes": 120,
        "test_script": "python build_tools/github_actions/test_executable_scripts/test_hiptests.py",
        "platform": ["linux", "windows"],
        "total_shards": 4,  # Runs as 4 parallel jobs
    },
    "rocblas": {
        "job_name": "rocblas",
        "fetch_artifact_args": "--blas --tests",
        "timeout_minutes": 15,
        "test_script": "python build_tools/github_actions/test_executable_scripts/test_rocblas.py",
        "platform": ["linux", "windows"],
        "total_shards": 1,
    },
    "rccl": {
        "job_name": "rccl",
        "platform": ["linux"],
        "total_shards": 1,
        "multi_gpu": {"linux": ["gfx94X-dcgpu"]},  # Requires multi-GPU runner
    },
    # ... 25+ more test definitions
}
```

**Available tests:**

| Category | Tests |
|----------|-------|
| Core | hip-tests |
| BLAS | rocblas, hipblas, hipblaslt, rocroller |
| Solver | rocsolver, hipsolver |
| Sparse | rocsparse, hipsparse, hipsparselt |
| Primitives | rocprim, hipcub, rocthrust |
| FFT | rocfft, hipfft |
| Random | rocrand, hiprand |
| ML/DNN | miopen, hipdnn, miopenprovider, hipblasltprovider |
| Communication | rccl |
| Tools | rocprofiler_systems, aqlprofile, rocrtst, rocr-debug-agent |

**Smoke vs Full tests:**

| Test Type | When Used | Shards | Purpose |
|-----------|-----------|--------|---------|
| **Smoke** | PRs, quick validation | 1 shard only | Fast feedback (~minutes) |
| **Full** | Nightly, submodule changes | All shards | Comprehensive coverage (~hours) |

## How the Benchmark Matrix Works

> **Source file:** [`tests/extended_tests/benchmark/benchmark_test_matrix.py`](../../tests/extended_tests/benchmark/benchmark_test_matrix.py)

Benchmarks are **separate from regular tests** and only run on nightly builds.

```python
benchmark_matrix = {
    "rocblas_bench": {
        "job_name": "rocblas_bench",
        "fetch_artifact_args": "--blas --tests",
        "timeout_minutes": 90,
        "test_script": "python tests/extended_tests/benchmark/scripts/test_rocblas_benchmark.py",
        "platform": ["linux"],
        "total_shards": 1,
    },
    "hipblaslt_bench": { ... },
    "rocsolver_bench": { ... },
    "rocrand_bench": { ... },
    "rocfft_bench": { ... },
}
```

**Key differences from regular tests:**
- Run on dedicated benchmark runners (multi-GPU machines)
- Never shard (always `total_shards: 1`)
- Only run on nightly, never on PRs

**How benchmarks are selected:**

```python
if is_benchmark_workflow:
    selected_matrix = benchmark_matrix  # Use benchmark tests
else:
    selected_matrix = test_matrix       # Use regular tests
```

## Related Documentation

- [Workflow Call Chains](workflow_call_chains.md) - Quick reference for all workflow call hierarchies
- [Release and Nightly Builds](release_and_nightly_builds.md) - Package versioning and S3 structure
- [CI Behavior Manipulation](ci_behavior_manipulation.md) - PR labels and CI controls
