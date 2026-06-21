# Test Harness Architecture

This document explains TheRock's generic, extensible test harness design and infrastructure architecture.

## Table of Contents

1. [Overview](#overview)
2. [Design Principles](#design-principles)
3. [Architecture Diagrams](#architecture-diagrams)
4. [Component Responsibilities](#component-responsibilities)
5. [Adding New Components](#adding-new-components)

---

## Overview

TheRock uses a **label-based, generic test harness** that separates concerns between:

- **Test Authors**: Define tests and categorize them with labels
- **Infrastructure Teams**: Manage runner pools independently
- **GitHub Actions**: Automatically dispatches matrix jobs to available runners

This design means **developers don't manage job dispatching** - they just write tests following simple labeling conventions.

---

## Design Principles

### 1. Generic & Extensible

- **One test runner** (`test_runner.py`) works for all components
- Components follow a **CTest labeling contract** (no component-specific code needed)
- New components added by configuration only (no code changes to runner)

### 2. Label-Based Configuration

- **Test categories**: `quick`, `standard`, `comprehensive`, `full`
- **GPU architectures**: `ex_gpu_gfx1151`, `ex_gpu_gfx11X` (with wildcard matching)
- **Exclusions**: `quick_exclude`, `standard_exclude`

### 3. Infrastructure Independence

- **Runner pools configured separately** by infra teams (GitHub runner labels)
- **Test matrix** specifies requirements, not specific runners
- **GitHub Actions scheduler** handles job → runner allocation
- **Weighted load balancing** across runner pools (automatic distribution)

### 4. Developer Experience

- Write tests in component repository
- Add CTest labels following convention
- No infrastructure knowledge required
- No manual job dispatching

---

## Architecture Diagrams

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DEVELOPER WORKFLOW                               │
│                                                                          │
│  ┌──────────────────┐                                                   │
│  │  Component Repo  │  1. Write test                                    │
│  │  (rocm-libs)     │  2. Add CTest labels: "quick", "ex_gpu_gfx1151"  │
│  └────────┬─────────┘                                                   │
│           │                                                              │
│           │ (Tests integrated into TheRock build)                       │
│           ▼                                                              │
│  ┌──────────────────┐                                                   │
│  │   TheRock Repo   │  3. Configure in fetch_test_configurations.py    │
│  │                  │     - Add to test_matrix dict                     │
│  └────────┬─────────┘     - Specify shards, timeout, platforms          │
│           │                                                              │
└───────────┼──────────────────────────────────────────────────────────────┘
            │
            │ Push / PR trigger
            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      GITHUB ACTIONS WORKFLOW ENGINE                      │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │  fetch_test_configurations.py                                │      │
│  │  - Reads test_matrix                                          │      │
│  │  - Applies filters (PROJECTS_TO_TEST, AMDGPU_FAMILIES)       │      │
│  │  - Generates JSON matrix:                                     │      │
│  │    [                                                          │      │
│  │      {job_name: "rocblas", total_shards: 6, shard_arr: [...]}│      │
│  │      {job_name: "miopen",  total_shards: 4, shard_arr: [...]}│      │
│  │      ...                                                      │      │
│  │    ]                                                          │      │
│  └────────────────────────┬─────────────────────────────────────┘      │
│                           │                                             │
│                           ▼                                             │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │  Matrix Job Expansion                                         │      │
│  │                                                               │      │
│  │  rocblas × shards[1,2,3,4,5,6] = 6 parallel jobs            │      │
│  │  miopen  × shards[1,2,3,4]      = 4 parallel jobs            │      │
│  │  ...                                                          │      │
│  │                                                               │      │
│  │  Total: 50+ jobs ready to dispatch                           │      │
│  └────────────────────────┬─────────────────────────────────────┘      │
│                           │                                             │
└───────────────────────────┼─────────────────────────────────────────────┘
                            │
                            │ Job dispatch requests
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   GITHUB ACTIONS JOB SCHEDULER                           │
│                    (Automatic Load Balancing)                            │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │  Job Queue                                                    │      │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐ │      │
│  │  │ rocblas shard 1 │  │ miopen shard 1  │  │ hip-tests 1  │ │      │
│  │  │ needs: gfx1100  │  │ needs: gfx1100  │  │ needs: ...   │ │      │
│  │  │ runs-on:        │  │ runs-on:        │  │              │ │      │
│  │  │  - gpu-runner-1 │  │  - gpu-runner-2 │  │              │ │      │
│  │  └─────────────────┘  └─────────────────┘  └──────────────┘ │      │
│  │            │                    │                   │         │      │
│  └────────────┼────────────────────┼───────────────────┼─────────┘      │
│               │                    │                   │                 │
│               │  GitHub matches jobs to runners based on labels         │
│               ▼                    ▼                   ▼                 │
└─────────────────────────────────────────────────────────────────────────┘
                │                    │                   │
                ▼                    ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         RUNNER INFRASTRUCTURE                            │
│              (Managed independently by infra teams)                      │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  Runner Pool: gfx1100                                        │       │
│  │  Labels: [linux, gpu, gfx1100-runner, pool-priority-high]  │       │
│  │                                                              │       │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │       │
│  │  │ Runner 1 │  │ Runner 2 │  │ Runner 3 │  │ Runner 4 │   │       │
│  │  │  (idle)  │  │  (busy)  │  │  (idle)  │  │  (busy)  │   │       │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  Runner Pool: gfx950                                         │       │
│  │  Labels: [linux, gpu, gfx950-runner, pool-priority-medium] │       │
│  │                                                              │       │
│  │  ┌──────────┐  ┌──────────┐                                │       │
│  │  │ Runner 1 │  │ Runner 2 │                                │       │
│  │  │  (idle)  │  │  (idle)  │                                │       │
│  │  └──────────┘  └──────────┘                                │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │  Runner Pool: Multi-GPU (gfx94X-dcgpu)                      │       │
│  │  Labels: [linux, gpu, multi-gpu, gfx94X-dcgpu-runner]      │       │
│  │                                                              │       │
│  │  ┌──────────┐  ┌──────────┐                                │       │
│  │  │ Runner 1 │  │ Runner 2 │                                │       │
│  │  │ 4x GPUs  │  │ 4x GPUs  │                                │       │
│  │  └──────────┘  └──────────┘                                │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                          │
│  Infra teams can:                                                       │
│  - Add/remove runners dynamically                                       │
│  - Change label weights for load balancing                              │
│  - Maintain pools independently                                         │
│  - No coordination with test authors required                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Test Execution Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SINGLE TEST JOB EXECUTION                             │
│                 (Example: rocblas, shard 2/6, gfx1100)                  │
└─────────────────────────────────────────────────────────────────────────┘

    Runner acquired: gfx1100-runner-3
           │
           ▼
    ┌─────────────────────┐
    │  Setup Environment  │
    │  - Fetch artifacts  │
    │  - Install deps     │
    │  - Setup venv       │
    └──────────┬──────────┘
               │
               ▼
    ┌─────────────────────┐
    │  Health Checks      │
    │  - Driver status    │
    │  - GPU visibility   │
    │  - rocm-smi         │
    └──────────┬──────────┘
               │
               ▼
    ┌─────────────────────────────────────────────────────────────┐
    │  Execute Test Script: test_runner.py                        │
    │                                                              │
    │  1. Read env vars:                                          │
    │     - TEST_COMPONENT=rocblas                                │
    │     - TEST_TYPE=standard                                    │
    │     - AMDGPU_FAMILIES=gfx1100                              │
    │     - SHARD_INDEX=2, TOTAL_SHARDS=6                        │
    │                                                              │
    │  2. Discover GPU labels:                                    │
    │     $ ctest --print-labels --test-dir ./build/bin/rocblas  │
    │     Found: [ex_gpu_gfx1100, ex_gpu_gfx110X, quick,         │
    │             standard, standard_exclude, ...]                │
    │                                                              │
    │  3. Match GPU architecture:                                 │
    │     Current: gfx1100                                        │
    │     Available: [gfx1100, gfx110X, gfx11X]                  │
    │     Best match: gfx1100 (exact match)                       │
    │                                                              │
    │  4. Build CTest command:                                    │
    │     ctest \                                                 │
    │       -L standard \              # Category label           │
    │       -L ex_gpu_gfx1100 \        # GPU-specific tests       │
    │       --parallel 8 \             # Run 8 tests concurrently │
    │       --output-on-failure \      # Show failed test output  │
    │       --test-dir ./build/bin/rocblas \                      │
    │       --tests-information 2,6    # Shard 2 of 6            │
    │                                                              │
    │  5. Execute with environment:                               │
    │     ROCM_PATH=/path/to/rocm                                │
    │     GTEST_SHARD_INDEX=1 (0-indexed)                        │
    │     GTEST_TOTAL_SHARDS=6                                   │
    └──────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
                   ┌───────────────┐
                   │  Test Results │
                   │  - Pass/Fail  │
                   │  - Logs       │
                   │  - Metrics    │
                   └───────────────┘
```

### Label-Based Test Selection

```
Component Test Suite Layout (Example: rocBLAS)
──────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────┐
│                       rocBLAS Test Suite                         │
│                  (Defined in CMakeLists.txt)                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ├── Categories
                              │   ├── quick (smoke tests, ~5 min)
                              │   ├── standard (pre-commit, ~30 min)
                              │   ├── comprehensive (~2 hours)
                              │   └── full (everything)
                              │
                              ├── GPU Architectures
                              │   ├── ex_gpu_gfx1100 (tests for gfx1100)
                              │   ├── ex_gpu_gfx110X (tests for gfx110X family)
                              │   ├── ex_gpu_gfx11X  (tests for gfx11XX family)
                              │   ├── ex_gpu_gfx950  (tests for gfx950)
                              │   └── ex_gpu_gfx94X  (tests for gfx94X family)
                              │
                              └── Exclusions
                                  ├── quick_exclude (skip in quick mode)
                                  ├── standard_exclude (skip in standard mode)
                                  └── windows_exclude (skip on Windows)

Example Test Definition (CMake):
────────────────────────────────

add_test(NAME rocblas_gemm_gfx1100_standard
         COMMAND rocblas_test_gemm)
set_tests_properties(rocblas_gemm_gfx1100_standard PROPERTIES
    LABELS "standard;ex_gpu_gfx1100"
    TIMEOUT 600
)

Test Selection Logic (test_runner.py):
───────────────────────────────────────

Input:
  TEST_TYPE = "standard"
  AMDGPU_FAMILIES = "gfx1100"

Step 1: Discover available labels
  $ ctest --print-labels
  → [quick, standard, ex_gpu_gfx1100, ex_gpu_gfx110X, ...]

Step 2: Match GPU architecture
  Current GPU: gfx1100
  Available:   [gfx1100, gfx110X, gfx11X]
  Match:       gfx1100 (exact match wins)

Step 3: Build CTest filter
  ctest -L standard -L ex_gpu_gfx1100

Result:
  ✓ Runs: Tests with BOTH "standard" AND "ex_gpu_gfx1100" labels
  ✗ Skips: Tests without "standard" label
  ✗ Skips: Tests with "ex_gpu_gfx950" label (different GPU)
  ✗ Skips: Tests with "standard_exclude" label

Wildcard Matching Example:
──────────────────────────

Scenario: Running on gfx1151, but component only has gfx115X label

Input:
  Current GPU: gfx1151
  Available labels: [ex_gpu_gfx115X, ex_gpu_gfx11X]

Matching algorithm (most specific first):
  1. Try exact: gfx1151 → NOT FOUND
  2. Try gfx115X → FOUND ✓
  3. (Would try gfx11X if above failed)

Result:
  ctest -L standard -L ex_gpu_gfx115X

This allows components to define tests for GPU families without
enumerating every single variant.
```

### Infrastructure Scaling & Load Balancing

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    INFRASTRUCTURE TEAM VIEW                              │
│               (Runner Management - Independent of Tests)                 │
└─────────────────────────────────────────────────────────────────────────┘

Current Runner Allocation:
──────────────────────────

┌──────────────────────────────────────────────────────────────┐
│  GPU Architecture: gfx1100                                    │
│  Weekly job volume: ~500 jobs                                │
│  Peak concurrency: 20 jobs                                   │
│                                                               │
│  Runner Pool Configuration:                                  │
│  ┌────────────────────────────────────────────────────┐     │
│  │  runner-group: gfx1100-pool-A                      │     │
│  │  labels: [linux, gpu, gfx1100-runner]             │     │
│  │  weight: 70%                                        │     │
│  │  machines: 12 runners                              │     │
│  └────────────────────────────────────────────────────┘     │
│                                                               │
│  ┌────────────────────────────────────────────────────┐     │
│  │  runner-group: gfx1100-pool-B                      │     │
│  │  labels: [linux, gpu, gfx1100-runner]             │     │
│  │  weight: 30%                                        │     │
│  │  machines: 5 runners                               │     │
│  └────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  GPU Architecture: gfx950                                     │
│  Weekly job volume: ~200 jobs                                │
│  Peak concurrency: 8 jobs                                    │
│                                                               │
│  Runner Pool Configuration:                                  │
│  ┌────────────────────────────────────────────────────┐     │
│  │  runner-group: gfx950-pool                         │     │
│  │  labels: [linux, gpu, gfx950-runner]              │     │
│  │  weight: 100%                                       │     │
│  │  machines: 6 runners                               │     │
│  └────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘

Scaling Scenario: New Hardware Arrives
───────────────────────────────────────

Week 1: Initial allocation
  gfx1100-pool-A: 12 runners (70% weight)
  gfx1100-pool-B: 5 runners  (30% weight)

Week 4: Pool-B gets 5 more machines
  Action by infra team:
  1. Add 5 new runners to pool-B
  2. Update weight: Pool-A=50%, Pool-B=50%
  3. No changes to test configurations needed
  4. GitHub automatically redistributes load

Week 8: Pool-A hardware refresh
  Action by infra team:
  1. Drain pool-A (set weight=0%)
  2. Update machines
  3. Restore pool-A (set weight=70%)
  4. Tests continue running on pool-B during refresh

Developers see:
  ✓ Tests keep running
  ✓ Same test configurations
  ✓ Same results
  ✗ Zero awareness of infrastructure changes

Load Balancing Algorithm:
──────────────────────────

When test job requests "gfx1100-runner":

┌─────────────────────────────────────────────────────────────┐
│  fetch_test_configurations.py                               │
│                                                              │
│  Per-component random selection based on weights:           │
│                                                              │
│  for component in all_components:                           │
│      if test_runs_on_labels:                                │
│          # Weighted random selection                        │
│          runner = select_weighted_label(                    │
│              labels=[                                       │
│                  {"label": "gfx1100-pool-A", "weight": 70}, │
│                  {"label": "gfx1100-pool-B", "weight": 30}  │
│              ],                                             │
│              seed=component.job_name  # Deterministic       │
│          )                                                  │
│          component["test_runner"] = runner                  │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
        Result distribution (50 jobs):
        ┌──────────────────────────────────┐
        │ ~35 jobs → gfx1100-pool-A (70%) │
        │ ~15 jobs → gfx1100-pool-B (30%) │
        └──────────────────────────────────┘

Benefits:
  ✓ Automatic load distribution
  ✓ Infra teams control allocation via weights
  ✓ No manual job assignment
  ✓ Easy to add/remove capacity
  ✓ Gradual migration between pools
```

### Matrix Job Generation & Dispatch

```
┌─────────────────────────────────────────────────────────────────────────┐
│              HOW MATRIX JOBS ARE GENERATED & DISPATCHED                  │
└─────────────────────────────────────────────────────────────────────────┘

Step 1: Test Configuration (fetch_test_configurations.py)
──────────────────────────────────────────────────────────

Input environment:
  PROJECTS_TO_TEST = "rocblas,miopen"
  AMDGPU_FAMILIES = "gfx1100"
  TEST_TYPE = "standard"

Processing:
  ┌────────────────────────────────────────────────────────┐
  │  for component in ["rocblas", "miopen"]:              │
  │      config = test_matrix[component]                   │
  │      total_shards = config.total_shards_dict["linux"]  │
  │      shard_arr = [1, 2, 3, ..., total_shards]         │
  │      components.append({                               │
  │          "job_name": component,                        │
  │          "total_shards": total_shards,                 │
  │          "shard_arr": shard_arr,                       │
  │          "test_script": "python .../test_runner.py",   │
  │          "timeout_minutes": config.timeout,            │
  │          "test_runner": select_weighted_label(...)     │
  │      })                                                │
  └────────────────────────────────────────────────────────┘

Output (JSON to GitHub Actions):
  {
    "components": [
      {
        "job_name": "rocblas",
        "total_shards": 6,
        "shard_arr": [1, 2, 3, 4, 5, 6],
        "test_script": "python .../test_runner.py",
        "timeout_minutes": 288,
        "test_runner": "gfx1100-pool-A"
      },
      {
        "job_name": "miopen",
        "total_shards": 4,
        "shard_arr": [1, 2, 3, 4],
        "test_script": "python .../test_runner.py",
        "timeout_minutes": 120,
        "test_runner": "gfx1100-pool-B"
      }
    ]
  }

Step 2: GitHub Actions Matrix Expansion
────────────────────────────────────────

Workflow definition:
  jobs:
    test_component:
      strategy:
        matrix:
          shard: ${{ fromJSON(inputs.component).shard_arr }}
      runs-on: ${{ fromJSON(inputs.component).test_runner }}

Expansion for rocblas:
  ┌─────────────────────────────────────────────────────────┐
  │  Job: test_component (rocblas, shard 1/6)              │
  │  runs-on: gfx1100-pool-A                                │
  ├─────────────────────────────────────────────────────────┤
  │  Job: test_component (rocblas, shard 2/6)              │
  │  runs-on: gfx1100-pool-A                                │
  ├─────────────────────────────────────────────────────────┤
  │  Job: test_component (rocblas, shard 3/6)              │
  │  runs-on: gfx1100-pool-A                                │
  ├─────────────────────────────────────────────────────────┤
  │  ... (shards 4, 5, 6)                                   │
  └─────────────────────────────────────────────────────────┘

Expansion for miopen:
  ┌─────────────────────────────────────────────────────────┐
  │  Job: test_component (miopen, shard 1/4)               │
  │  runs-on: gfx1100-pool-B                                │
  ├─────────────────────────────────────────────────────────┤
  │  Job: test_component (miopen, shard 2/4)               │
  │  runs-on: gfx1100-pool-B                                │
  ├─────────────────────────────────────────────────────────┤
  │  ... (shards 3, 4)                                      │
  └─────────────────────────────────────────────────────────┘

Total: 10 parallel jobs created automatically

Step 3: GitHub Actions Job Queue & Dispatch
────────────────────────────────────────────

GitHub Actions Scheduler:

  ┌───────────────────────────────────────────────────────┐
  │  Job Queue (all jobs waiting for runners)            │
  │                                                        │
  │  1. rocblas shard 1/6  → needs: gfx1100-pool-A       │
  │  2. rocblas shard 2/6  → needs: gfx1100-pool-A       │
  │  3. rocblas shard 3/6  → needs: gfx1100-pool-A       │
  │  4. rocblas shard 4/6  → needs: gfx1100-pool-A       │
  │  5. rocblas shard 5/6  → needs: gfx1100-pool-A       │
  │  6. rocblas shard 6/6  → needs: gfx1100-pool-A       │
  │  7. miopen shard 1/4   → needs: gfx1100-pool-B       │
  │  8. miopen shard 2/4   → needs: gfx1100-pool-B       │
  │  9. miopen shard 3/4   → needs: gfx1100-pool-B       │
  │  10. miopen shard 4/4  → needs: gfx1100-pool-B       │
  └───────────────────────────────────────────────────────┘
                    │
                    │ GitHub matches jobs to runners
                    ▼
  ┌───────────────────────────────────────────────────────┐
  │  Available Runners                                     │
  │                                                        │
  │  Pool A (12 runners with label gfx1100-pool-A):       │
  │    Runner-A1 (idle)   → Assigned: rocblas shard 1    │
  │    Runner-A2 (idle)   → Assigned: rocblas shard 2    │
  │    Runner-A3 (idle)   → Assigned: rocblas shard 3    │
  │    Runner-A4 (idle)   → Assigned: rocblas shard 4    │
  │    Runner-A5 (idle)   → Assigned: rocblas shard 5    │
  │    Runner-A6 (idle)   → Assigned: rocblas shard 6    │
  │    Runner-A7..A12 (idle or other jobs)                │
  │                                                        │
  │  Pool B (5 runners with label gfx1100-pool-B):        │
  │    Runner-B1 (idle)   → Assigned: miopen shard 1     │
  │    Runner-B2 (idle)   → Assigned: miopen shard 2     │
  │    Runner-B3 (idle)   → Assigned: miopen shard 3     │
  │    Runner-B4 (idle)   → Assigned: miopen shard 4     │
  │    Runner-B5 (idle or other jobs)                     │
  └───────────────────────────────────────────────────────┘

Key Points:
  ✓ Developers specify: "I need a gfx1100 runner"
  ✓ GitHub Actions handles: Job → Runner matching
  ✓ Infra teams control: Which runners have which labels
  ✓ Load balancing: Automatic based on weights
  ✓ Scaling: Add runners without changing workflows
  ✓ No manual dispatch: Everything automatic
```

---

## Component Responsibilities

### Test Authors (Component Developers)

**Responsibilities:**
- Write tests in component repository
- Add CTest labels following convention
- Define test categories (quick/standard/comprehensive/full)
- Mark GPU-specific tests with `ex_gpu_*` labels

**What they DON'T manage:**
- Runner infrastructure
- Job scheduling
- Load balancing
- Hardware availability

**Example:**
```cmake
# In rocBLAS/test/CMakeLists.txt

add_test(NAME rocblas_gemm_quick
         COMMAND rocblas_test_gemm --quick)
set_tests_properties(rocblas_gemm_quick PROPERTIES
    LABELS "quick"  # Category
)

add_test(NAME rocblas_gemm_gfx1100
         COMMAND rocblas_test_gemm --full)
set_tests_properties(rocblas_gemm_gfx1100 PROPERTIES
    LABELS "standard;ex_gpu_gfx1100"  # Category + GPU-specific
    TIMEOUT 3600
)
```

### TheRock Integration Team

**Responsibilities:**
- Add component to `test_matrix` in `fetch_test_configurations.py`
- Configure sharding, timeouts, platform support
- Map component job names to directories

**What they DON'T manage:**
- Individual test definitions
- Runner pools
- Infrastructure scaling

**Example:**
```python
# In build_tools/github_actions/fetch_test_configurations.py

test_matrix = {
    "my-new-component": {
        "job_name": "my-new-component",
        "fetch_artifact_args": "--my-component --tests",
        "timeout_minutes": 60,
        "test_script": f"python {_get_script_path('test_runner.py')}",
        "platform": ["linux", "windows"],
        "total_shards_dict": {
            "linux": 4,
            "windows": 2,
        },
    }
}
```

### Infrastructure Teams

**Responsibilities:**
- Provision and maintain runner hardware
- Configure runner labels and pools
- Adjust load balancing weights
- Scale capacity up/down
- Monitor runner health

**What they DON'T manage:**
- Test definitions
- Test categories
- Job configurations
- Which tests run on which hardware

**Example:**
```bash
# Register runner with GitHub

./config.sh \
  --url https://github.com/ROCm/TheRock \
  --token ${RUNNER_TOKEN} \
  --labels linux,gpu,gfx1100-runner,gfx1100-pool-A \
  --name gfx1100-runner-01

# Update weight in amdgpu_family_matrix.py
"gfx1100": {
    "linux": {
        "test-runs-on-labels": [
            {"label": "gfx1100-pool-A", "weight": 70},
            {"label": "gfx1100-pool-B", "weight": 30}
        ]
    }
}
```

### GitHub Actions (Automatic)

**Handles:**
- Matrix job expansion
- Job queue management
- Runner selection based on labels
- Job dispatch and scheduling
- Retry logic
- Parallel execution

**No human intervention needed**

---

## Adding New Components

### Step 1: Component Repository (Test Author)

Define tests with CTest labels:

```cmake
# In your_component/test/CMakeLists.txt

enable_testing()

# Quick smoke test (all GPUs)
add_test(NAME your_component_smoke
         COMMAND your_component_test --smoke)
set_tests_properties(your_component_smoke PROPERTIES
    LABELS "quick"
    TIMEOUT 300
)

# Standard test for gfx1100 family
add_test(NAME your_component_standard_gfx1100
         COMMAND your_component_test --standard)
set_tests_properties(your_component_standard_gfx1100 PROPERTIES
    LABELS "standard;ex_gpu_gfx110X"
    TIMEOUT 1800
)

# Comprehensive test for all gfx11XX GPUs
add_test(NAME your_component_comprehensive_gfx11xx
         COMMAND your_component_test --comprehensive)
set_tests_properties(your_component_comprehensive_gfx11xx PROPERTIES
    LABELS "comprehensive;ex_gpu_gfx11X"
    TIMEOUT 3600
)

# Full test suite (no GPU restrictions)
add_test(NAME your_component_full
         COMMAND your_component_test --all)
set_tests_properties(your_component_full PROPERTIES
    LABELS "full"
    TIMEOUT 7200
)
```

### Step 2: TheRock Configuration

#### 2a. Add directory mapping (test_runner.py):

```python
COMPONENT_DIR_MAPPING = {
    # ... existing mappings ...
    "your-component": "YourComponent",  # job_name → directory
}
```

#### 2b. Add to test matrix (fetch_test_configurations.py):

```python
test_matrix = {
    # ... existing components ...
    "your-component": {
        "job_name": "your-component",
        "fetch_artifact_args": "--your-component --tests",
        "timeout_minutes": 90,
        "test_script": f"python {_get_script_path('test_runner.py')}",
        "platform": ["linux", "windows"],
        "total_shards_dict": {
            "linux": 2,
            "windows": 1,
        },
    },
}
```

### Step 3: Test Locally

```bash
# Build TheRock with your component
cmake -B build -GNinja \
  -DTHEROCK_ENABLE_YOUR_COMPONENT=ON \
  -DTHEROCK_AMDGPU_FAMILIES=gfx1100

ninja -C build your-component

# Run tests via the generic runner
export TEST_COMPONENT=your-component
export TEST_TYPE=quick
export AMDGPU_FAMILIES=gfx1100
export THEROCK_BIN_DIR=./build/dist/rocm/bin
export SHARD_INDEX=1
export TOTAL_SHARDS=1

python build_tools/github_actions/test_executable_scripts/test_runner.py
```

### Step 4: Verify in CI

Push a PR and check:
- [ ] Component appears in test matrix
- [ ] Jobs are created for each shard
- [ ] Jobs dispatch to appropriate runners
- [ ] Tests execute with correct labels
- [ ] Results reported correctly

**That's it!** No infrastructure setup, no runner management, no job dispatching code.

---

## Summary

### Key Benefits

1. **Separation of Concerns**
   - Test authors focus on tests
   - Infra teams focus on hardware
   - GitHub handles scheduling

2. **Extensibility**
   - Add components via configuration
   - No code changes to test runner
   - Labels provide flexibility

3. **Scalability**
   - Parallel execution via sharding
   - Dynamic runner allocation
   - Automatic load balancing

4. **Maintainability**
   - Single generic test runner
   - Consistent conventions
   - Easy to debug

5. **Developer Experience**
   - Write tests, add labels, done
   - No infrastructure knowledge needed
   - Fast feedback via sharding

### Design Patterns

- **Convention over Configuration**: Follow labeling convention, get automation
- **Declarative over Imperative**: Declare what you need, not how to get it
- **Separation of Concerns**: Clear boundaries between roles
- **Composition over Inheritance**: Combine labels to build test suites
- **Fail Fast**: Health checks before tests, clear error messages

---

## References

- [test_runner.py](../../build_tools/github_actions/test_executable_scripts/test_runner.py) - Generic test runner implementation
- [fetch_test_configurations.py](../../build_tools/github_actions/fetch_test_configurations.py) - Test matrix configuration
- [test_component.yml](../../.github/workflows/test_component.yml) - Reusable test workflow
- [README.md](../../build_tools/github_actions/test_executable_scripts/README.md) - Test runner documentation
