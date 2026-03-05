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

> **Source:** [`build_tools/github_actions/new_amdgpu_family_matrix.py`](../../build_tools/github_actions/new_amdgpu_family_matrix.py) lines 62-76

| Event Type | When It Runs | GPU Families Included | Test Type | Purpose |
|------------|--------------|----------------------|-----------|---------|
| **Presubmit** | Every pull request (`pull_request`) | `gfx94X-dcgpu`, `gfx110X-all`, `gfx1151`, `gfx120X-all` | Smoke (1 shard) | Fast feedback for contributors |
| **Postsubmit** | Every push to `main` (`push`) | Presubmit families + `gfx950-dcgpu` | Smoke (1 shard) | Validate merged code on more hardware |
| **Nightly** | 2 AM UTC daily (`schedule`) | All families: presubmit + postsubmit + `gfx90X-dcgpu`, `gfx101X-dgpu`, `gfx103X-dgpu`, `gfx1150`, `gfx1152`, `gfx1153` | Full (all shards) | Comprehensive coverage including older/experimental GPUs |

**Why this matters:** Not all GPU hardware is available in large quantities. Presubmit runs on the most common GPUs for quick feedback. Nightly runs on everything to catch architecture-specific issues.

**Test type determination** ([`configure_ci.py:637-678`](../../build_tools/github_actions/configure_ci.py)):
- **Default:** `smoke` tests (1 shard per test, fast feedback)
- **Nightly (schedule):** `full` tests (all shards, comprehensive coverage)
- **Submodule changed:** `full` tests (source code changes require thorough testing)
- **Test labels specified:** `full` tests (explicit request for specific tests)

## How the GPU Matrix Is Created

> **Source file:** [`build_tools/github_actions/new_amdgpu_family_matrix.py`](../../build_tools/github_actions/new_amdgpu_family_matrix.py)

This file is the **source of truth** for which GPUs exist and how to test them.

**Event-based grouping** (lines 62-76):

```python
amdgpu_family_predefined_groups = {
    # Presubmit: runs on pull_request triggers
    "amdgpu_presubmit": ["gfx94X-dcgpu", "gfx110X-all", "gfx1151", "gfx120X-all"],

    # Postsubmit: runs on push triggers (commits to main)
    "amdgpu_postsubmit": ["gfx950-dcgpu"],

    # Nightly: runs on schedule triggers (2 AM UTC daily)
    "amdgpu_nightly": [
        "gfx90X-dcgpu",   # MI200 series (gfx90a)
        "gfx101X-dgpu",   # RDNA1
        "gfx103X-dgpu",   # RDNA2
        "gfx1150",        # Strix Point variants
        "gfx1152",
        "gfx1153",
    ],
}
```

**Per-GPU configuration** (`amdgpu_family_info_matrix_all`, lines 105-511) defines build/test details for each GPU:

```python
amdgpu_family_info_matrix_all = {
    "gfx94X": {  # MI325X family
        "dcgpu": {
            "linux": {
                "build": {
                    "build_variants": ["release", "asan"],  # Build configurations
                },
                "test": {
                    "run_tests": True,
                    "runs_on": {
                        "test": "linux-mi325-1gpu-ossci-rocm-frac",      # Single-GPU tests
                        "test-multi-gpu": "linux-mi325-8gpu-ossci-rocm", # Multi-GPU tests (RCCL)
                        "benchmark": "linux-mi325-8gpu-ossci-rocm",      # Benchmarks
                    },
                    "fetch-gfx-targets": ["gfx942"],  # Specific GPU chips to fetch artifacts for
                },
                "release": {
                    "push_on_success": True,                  # Push releases to S3
                    "bypass_tests_for_releases": False,      # Don't skip tests
                },
            },
            "windows": {
                "build": {"build_variants": ["release"]},
                "test": {"run_tests": False},  # Windows tests disabled for gfx94X
                "release": {"push_on_success": False},
            },
        },
    },
    "gfx950": {  # MI355X family
        "dcgpu": {
            "linux": {
                "build": {"build_variants": ["release", "asan"]},
                "test": {
                    "run_tests": True,
                    "runs_on": {"test": "linux-mi355-1gpu-ossci-rocm"},
                    "fetch-gfx-targets": ["gfx950"],
                },
                "release": {"push_on_success": False},  # Not released yet
            },
        },
    },
    "gfx110X": {  # RDNA3
        "all": {
            "linux": {
                "test": {
                    "run_tests": False,  # TODO(#2740): Re-enable once amdsmi test is fixed
                    "sanity_check_only_for_family": True,  # Only run sanity checks
                },
                "release": {"bypass_tests_for_releases": True},
            },
            "windows": {
                "test": {
                    "run_tests": True,
                    "runs_on": {"test": "windows-gfx110X-gpu-rocm"},
                    "sanity_check_only_for_family": True,
                },
            },
        },
    },
    "gfx120X": {  # RDNA4
        "all": {
            "linux": {
                "test": {
                    "run_tests": True,
                    "runs_on": {"test": "linux-gfx120X-gpu-rocm"},
                    "fetch-gfx-targets": ["gfx1200", "gfx1201"],  # Multiple specific GPU chips
                    "sanity_check_only_for_family": True,
                },
            },
        },
    },
    # ... (more families: gfx1151, gfx1152, gfx1153, gfx90X, gfx101X, gfx103X)
}
```

**GPU family naming convention** ([`cmake/therock_amdgpu_targets.cmake`](../../cmake/therock_amdgpu_targets.cmake)):

GPU families use suffixes to indicate GPU categories:

| Suffix | Full Name | Description | Examples |
|--------|-----------|-------------|----------|
| `-dcgpu` | Datacenter GPU | Server GPUs with matrix cores, ECC memory | `gfx94X-dcgpu` (MI300X), `gfx950-dcgpu` (MI350X) |
| `-dgpu` | Discrete GPU | Desktop/workstation GPUs | `gfx110X-dgpu` (RX 7900), `gfx103X-dgpu` (RX 6800) |
| `-igpu` | Integrated GPU | Laptop/APU integrated graphics | `gfx110X-igpu` (Radeon 780M), `gfx115X-igpu` (Strix Point) |
| `-all` | All variants | Includes all GPU types in that architecture family | `gfx110X-all` (all RDNA3), `gfx120X-all` (all RDNA4) |

**How families work** (lines 47-73):

```cmake
# Single specific GPU chip
therock_add_amdgpu_target(gfx942 "MI300A/MI300X CDNA"
    FAMILY dcgpu-all gfx94X-all gfx94X-dcgpu)
    # This chip belongs to 3 families:
    # - dcgpu-all (all datacenter GPUs)
    # - gfx94X-all (all gfx94X variants)
    # - gfx94X-dcgpu (only datacenter gfx94X)

# Multiple chips in one family
therock_add_amdgpu_target(gfx1100 "AMD RX 7900 XTX"
    FAMILY dgpu-all gfx110X-all gfx110X-dgpu)
therock_add_amdgpu_target(gfx1103 "AMD Radeon 780M Laptop iGPU"
    FAMILY igpu-all gfx110X-all gfx110X-igpu)
    # Both belong to gfx110X-all, but different GPU types
```

**Why this matters for CI:**
- `gfx94X-dcgpu` → Tests only MI300 datacenter GPUs (most important for ROCm)
- `gfx110X-all` → Tests all RDNA3 (desktop + laptop), broader hardware coverage
- `gfx103X-dgpu` → Tests only discrete RDNA2, excludes integrated GPUs

**Build variants** (lines 78-102) define different build configurations:

| Variant | Platform | Preset | Purpose |
|---------|----------|--------|---------|
| `release` | Linux | `linux-release` | Standard optimized build |
| `asan` | Linux | `linux-release-asan` | AddressSanitizer (memory error detection) |
| `release` | Windows | `windows-release` | Standard Windows build |

**How families are selected** ([`configure_ci.py`](../../build_tools/github_actions/configure_ci.py)):

```
pull_request  → presubmit families only (4 GPUs)
push to main  → presubmit + postsubmit families (5 GPUs)
schedule      → presubmit + postsubmit + nightly families (11 GPUs total)
```

**Key flags:**
- `run_tests: False` - Build only, skip tests (used for GPUs without test runners)
- `sanity_check_only_for_family: True` - Run minimal validation tests only
- `bypass_tests_for_releases: True` - Skip tests when creating releases (build-only)
- `push_on_success: True` - Upload to S3 release buckets after successful build

## How the Test Matrix Is Created

> **Source file:** [`build_tools/github_actions/fetch_test_configurations.py`](../../build_tools/github_actions/fetch_test_configurations.py)

This file defines the `test_matrix` dictionary listing **what tests exist** and **how to run them**.

### Test Matrix Structure

Tests are defined with platform support, sharding, and artifact dependencies:

```python
test_matrix = {
    "hip-tests": {
        "job_name": "hip-tests",
        "fetch_artifact_args": "--tests",                    # Artifacts needed
        "timeout_minutes": 120,
        "test_script": "python build_tools/github_actions/test_executable_scripts/test_hiptests.py",
        "platform": ["linux", "windows"],                     # Supported platforms
        "total_shards_dict": {                                # Shards per platform
            "linux": 4,    # Linux: split into 4 parallel jobs
            "windows": 4,  # Windows: split into 4 parallel jobs
        },
    },
    "rocblas": {
        "job_name": "rocblas",
        "fetch_artifact_args": "--blas --tests",
        "timeout_minutes": 15,
        "test_script": "python build_tools/github_actions/test_executable_scripts/test_rocblas.py",
        "platform": ["linux", "windows"],
        "total_shards_dict": {
            "linux": 1,    # No sharding needed (fast test)
            "windows": 1,
        },
    },
    "rocroller": {
        "job_name": "rocroller",
        "fetch_artifact_args": "--blas --tests",
        "platform": ["linux"],
        "total_shards_dict": {"linux": 5},  # 5-way split for Linux
        "exclude_family": {                  # Architecture exclusions
            "linux": ["gfx1150", "gfx1151", "gfx1152", "gfx1153"],  # Not supported on gfx115X
        },
    },
    "rccl": {
        "job_name": "rccl",
        "fetch_artifact_args": "--rccl --tests",
        "platform": ["linux"],
        "total_shards_dict": {"linux": 1},
        "multi_gpu": {                        # Requires multi-GPU runners
            "linux": ["gfx94X-dcgpu"]         # Only gfx94X has multi-GPU runners
        },
    },
    # ... 30+ more test definitions (lines 38-451)
}
```

### Available Tests (Grouped by Category)

**Current test matrix** (as of new_amdgpu_family_matrix.py integration):

| Category | Tests | Linux Shards | Windows Shards | Notes |
|----------|-------|--------------|----------------|-------|
| **Core** | hip-tests | 4 | 4 | HIP runtime tests |
| **BLAS** | rocblas, hipblas, hipblaslt, rocroller | 1,1,6,5 | 1,1,1,5 | Matrix operations |
| **Solver** | rocsolver, hipsolver | 1,1 | -,1 | Linear algebra |
| **Sparse** | rocsparse, hipsparse, hipsparselt | 1,1,1 | 1,1,- | Sparse matrices |
| **Primitives** | rocprim, hipcub, rocthrust | 2,1,1 | 2,1,1 | GPU primitives |
| **FFT** | rocfft, hipfft | 1,2 | -,2 | Fast Fourier Transform |
| **Random** | rocrand, hiprand | 1,1 | 1,1 | Random number generation |
| **ML/DNN** | miopen, hipdnn, hipdnn-samples, miopenprovider, hipblasltprovider | 4,1,1,1,1 | 4,1,1,1,1 | Deep learning |
| **Communication** | rccl | 1 | - | Multi-GPU communication (Linux only) |
| **Profiling** | rocprofiler_systems, rocprofiler-compute, aqlprofile, rocrtst | 1,2,1,1 | - | Performance tools |
| **Debug** | rocgdb, rocr-debug-agent | 1,1 | - | Debugging tools |
| **Other** | rocwmma, libhipcxx_hipcc, libhipcxx_hiprtc, hipdnn_install | 4,1,1,1 | 2,-,-,1 | Specialized tests |

**Total:** 31 test suites, 58+ parallel test jobs (full runs)

### Smoke vs Full Tests: How Sharding Works

**Test type is determined by [`configure_ci.py:637-678`](../../build_tools/github_actions/configure_ci.py):**

| Trigger | Test Type | Sharding Behavior | Example |
|---------|-----------|-------------------|---------|
| `pull_request` | `smoke` | 1 shard only (first shard) | hip-tests runs 1 job instead of 4 |
| `push` | `smoke` | 1 shard only | Fast feedback on main branch |
| `schedule` (nightly) | `full` | All shards | hip-tests runs all 4 shards in parallel |
| Submodule changed | `full` | All shards | Source changes need thorough testing |
| Test labels specified | `full` | All shards (for labeled tests only) | Explicit test request |

**Implementation** ([`fetch_test_configurations.py:514-518`](../../build_tools/github_actions/fetch_test_configurations.py)):

```python
if test_type == "smoke":
    job_config_data["total_shards"] = 1       # Override to 1 shard
    job_config_data["shard_arr"] = [1]        # Run only first shard
else:  # test_type == "full"
    job_config_data["shard_arr"] = [1, 2, 3, 4]  # Run all shards
```

**Examples:**

```
hip-tests (total_shards_dict = {"linux": 4}):
  - Smoke test:  Runs 1 job  (shard 1 of 4) → ~30 min
  - Full test:   Runs 4 jobs (shards 1,2,3,4 in parallel) → ~30 min (4x throughput)

rocroller (total_shards_dict = {"linux": 5}):
  - Smoke test:  Runs 1 job  (shard 1 of 5) → ~12 min
  - Full test:   Runs 5 jobs (shards 1,2,3,4,5 in parallel) → ~12 min (5x throughput)
```

**Why sharding matters:**
- **Smoke tests:** Run fastest subset for quick PR feedback (1 shard = 1/Nth of tests)
- **Full tests:** Run complete test suite across all shards for comprehensive coverage
- **Parallelism:** Shards run simultaneously on separate runners, keeping wall-clock time constant

### Multi-GPU Tests

Some tests require multiple GPUs and use specialized runners:

```python
"multi_gpu": {
    "linux": ["gfx94X-dcgpu"]  # Only gfx94X has multi-GPU runners
}
```

**When multi-GPU is required** ([`fetch_test_configurations.py:520-547`](../../build_tools/github_actions/fetch_test_configurations.py)):
- Uses `test-runs-on-multi-gpu` runner instead of `test-runs-on`
- Example: `gfx94X`: `linux-mi325-8gpu-ossci-rocm` (8-GPU runner)
- If architecture doesn't have multi-GPU runner, test is skipped

**Tests requiring multi-GPU:**
- `rccl` - Multi-GPU communication library (only runs on gfx94X-dcgpu)

### Event-Based Test Control: How Tests Are Selected

**IMPORTANT:** Tests themselves do NOT have event-specific flags (no "only run on nightly" option in test_matrix). Event-based control happens through THREE mechanisms:

#### 1. GPU Family Selection (Primary Control)

Events control WHICH GPU families run, not which tests run:

```python
# From new_amdgpu_family_matrix.py:62-76
amdgpu_family_predefined_groups = {
    "amdgpu_presubmit": ["gfx94X-dcgpu", "gfx110X-all", "gfx1151", "gfx120X-all"],
    "amdgpu_postsubmit": ["gfx950-dcgpu"],
    "amdgpu_nightly": ["gfx90X-dcgpu", "gfx101X-dgpu", "gfx103X-dgpu", "gfx1150", "gfx1152", "gfx1153"],
}
```

**Example flow:**
```
pull_request (presubmit):
  ├─ gfx94X-dcgpu  → runs ALL tests (hip-tests, rocblas, miopen, rccl, etc.)
  ├─ gfx110X-all   → runs ALL tests
  ├─ gfx1151       → runs ALL tests
  └─ gfx120X-all   → runs ALL tests

schedule (nightly):
  ├─ gfx94X-dcgpu  → runs ALL tests (same tests as presubmit)
  ├─ gfx110X-all   → runs ALL tests
  ├─ ... (all 11 GPU families)
  └─ gfx1153       → runs ALL tests
```

#### 2. Test Type (Smoke vs Full)

Events control HOW MUCH of each test runs (sharding):

```python
# From configure_ci.py:637-678
if event == "schedule":
    test_type = "full"    # All shards
else:
    test_type = "smoke"   # 1 shard only
```

**All tests run on all events, but with different completeness:**
- Presubmit: hip-tests (1/4 shards), rocblas (1/1 shard), miopen (1/4 shards)
- Nightly: hip-tests (4/4 shards), rocblas (1/1 shard), miopen (4/4 shards)

#### 3. Benchmarks (Separate Matrix)

Benchmarks are the ONLY tests with event-specific control - they use a completely separate matrix:

```python
# From fetch_test_configurations.py:464-473
if is_benchmark_workflow:
    selected_matrix = benchmark_matrix  # Separate matrix, only on nightly
else:
    selected_matrix = test_matrix       # Regular tests, all events
```

**Benchmark control:**
- Only runs when `IS_BENCHMARK_WORKFLOW=true`
- Set by nightly workflow, NOT by PR/push workflows
- Completely separate from regular test_matrix

#### 4. TEST_LABELS (Manual Override)

Users can manually run specific tests via PR labels or workflow_dispatch:

```python
# From fetch_test_configurations.py:493-497
if test_labels and key not in test_labels:
    logging.info(f"Excluding job {job_name} since it's not in the test labels")
    continue  # Skip this test
```

**Usage:**
- Add PR label: `/test rocblas hipblas` → only runs rocblas and hipblas
- Workflow dispatch: Set `LINUX_TEST_LABELS=["miopen"]` → only runs miopen
- When test_labels is set, `test_type` becomes `"full"` (all shards)

#### 5. Per-GPU Test Exclusions (exclude_family)

Individual tests can be excluded for specific GPU families:

```python
# From fetch_test_configurations.py:63-88
"rocroller": {
    "exclude_family": {
        "linux": ["gfx1150", "gfx1151", "gfx1152", "gfx1153"],  # Skip on gfx115X
        "windows": ["gfx1150", "gfx1151", "gfx1152", "gfx1153"],
    },
},
```

**Implementation** (lines 482-491):
```python
if (
    "exclude_family" in selected_matrix[key]
    and platform in selected_matrix[key]["exclude_family"]
    and amdgpu_families in selected_matrix[key]["exclude_family"][platform]
):
    logging.info(f"Excluding job {job_name} for platform {platform} and family {amdgpu_families}")
    continue  # Skip this test for this GPU family
```

**Usage:** When a GPU family runs (e.g., gfx1151 on nightly), rocroller is skipped but all other tests run.

**Summary: Event Control is Indirect**

You **CANNOT** make hip-tests run only on postsubmit events. Event control works at the GPU/sharding level:

1. ✅ **Per-event GPU selection:** Make a GPU family run only on certain events (via `amdgpu_family_predefined_groups`)
2. ✅ **Per-event sharding:** Make ALL tests run with different completeness (via `test_type` smoke/full)
3. ✅ **Per-GPU test exclusion:** Skip specific tests on specific GPU families (via `exclude_family`)
4. ✅ **Separate benchmarks:** Create benchmark-only tests (via `benchmark_matrix`)
5. ✅ **Manual test selection:** Run specific tests via labels (via `TEST_LABELS`)
6. ❌ **Per-test event control:** Make specific regular tests skip certain events - **NOT SUPPORTED**

**Example combinations:**
- ✅ "Run rocroller only on nightly" → Put `gfx115X` in nightly group + add `exclude_family` for it
- ✅ "Skip rocroller on gfx1151" → Use `exclude_family`
- ❌ "Run hip-tests only on nightly" → Cannot do this (hip-tests runs on all events where GPUs are available)

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
- [Test Filtering](test_filtering.md) - Test filter levels and gtest implementation (Note: Currently documents 4 levels [smoke, standard, nightly, full], but actual code only uses smoke/full as of new_amdgpu_family_matrix.py integration)
