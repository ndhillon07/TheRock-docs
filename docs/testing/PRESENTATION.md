---
marp: true
theme: default
paginate: true
---

# TheRock Test Harness
## Configuration-Driven Generic Testing

Simple, scalable, framework-agnostic test orchestration

---

# Core Concept

```
┌─────────────────────────────────────────────────┐
│  You Write          GitHub Actions Does         │
│  ─────────          ──────────────────          │
│  • Configuration    • Creates parallel jobs     │
│  • Test labels      • Dispatches to runners     │
│                     • Collects results          │
│                                                 │
│  No job dispatch code needed!                   │
└─────────────────────────────────────────────────┘
```

---

# The Problem We Solved

**Before:**
- 100 components = 100 custom test scripts
- Manual job dispatching
- Hard-coded runner assignments
- Difficult to add new components
- Infrastructure changes break tests

**After:**
- 1 generic test runner for all components
- Automatic job dispatching via labels
- Infrastructure scales independently
- Add component = add config entry
- Zero coordination between teams

---

# Architecture: Two Files, Two Purposes

| File | Purpose | Managed By |
|------|---------|------------|
| **fetch_test_configurations.py** | **WHAT to test** | Test Engineers |
| **amdgpu_family_matrix.py** | **WHERE to test** | Infrastructure Team |

**Key Insight:** Test engineers and infrastructure teams work independently!

---

# File 1: WHAT to Test

**fetch_test_configurations.py**

```python
test_matrix = {
    "rocblas": {
        "job_name": "rocblas",
        "timeout_minutes": 288,
        "total_shards_dict": {
            "linux": 6,      # ← Creates 6 parallel jobs
        },
        "test_script": "python .../test_runner.py",
    },
}
```

**Defines:** Components, sharding, timeouts, test scripts

---

# File 2: WHERE to Test

**amdgpu_family_matrix.py**

```python
"gfx1100": {
    "linux": {
        "test-runs-on-labels": [
            {"label": "gfx1100-pool-A", "weight": 0.70},
            {"label": "gfx1100-pool-B", "weight": 0.30},
        ]
    }
}
```

**Defines:** GPU families, runner pools, load balancing weights

---

# How They Connect

```
Environment: AMDGPU_FAMILIES=gfx1100

┌────────────────────────────────────────────┐
│ fetch_test_configurations.py               │
│ "rocblas needs 6 shards for gfx1100"       │
└──────────────┬─────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────┐
│ amdgpu_family_matrix.py                    │
│ Lookup gfx1100 → Runner: pool-A (70%)      │
└──────────────┬─────────────────────────────┘
               │
               ▼
       GitHub Creates 6 Jobs
       All on: gfx1100-pool-A
```

---

# Complete Execution Flow

## Where Things Run

```
┌──────────────────────────────────────────────────┐
│ ubuntu-24.04 (GitHub-hosted, FREE)               │
│ • fetch_test_configurations.py                   │
│ • Generates JSON config                          │
│ • No GPU needed                                  │
└──────────────┬───────────────────────────────────┘
               │ JSON config
               ▼
┌──────────────────────────────────────────────────┐
│ YOUR GPU Runners (gfx1100-pool-A, etc.)          │
│ • test_runner.py                                 │
│ • Actual test execution                          │
│ • Uses GPU hardware                              │
└──────────────────────────────────────────────────┘
```

---

# Step 1: Configuration

**Runs on:** ubuntu-24.04 (GitHub-hosted)

```python
test_matrix = {
    "rocblas": {"total_shards_dict": {"linux": 6}},
    "miopen":  {"total_shards_dict": {"linux": 4}},
}
```

**Output:**
```json
{
  "components": [
    {"job_name": "rocblas", "shard_arr": [1,2,3,4,5,6]},
    {"job_name": "miopen", "shard_arr": [1,2,3,4]}
  ]
}
```

---

# Step 2: Matrix Expansion

**GitHub Actions reads JSON:**

```yaml
strategy:
  matrix:
    shard: [1, 2, 3, 4, 5, 6]  # For rocblas
```

**Creates 6 jobs automatically:**
- rocblas (shard 1/6)
- rocblas (shard 2/6)
- rocblas (shard 3/6)
- rocblas (shard 4/6)
- rocblas (shard 5/6)
- rocblas (shard 6/6)

---

# Step 3: Job Dispatch

**GitHub scheduler matches jobs to runners:**

```
Job Request: runs-on: gfx1100-pool-A
              ↓
Runner Pool: [Runner1, Runner2, Runner3, ...]
              ↓
GitHub finds idle runner
              ↓
Job assigned to Runner2
```

**Automatic!** No manual dispatch code.

---

# Step 4: Test Execution

**Runs on:** YOUR GPU runner (gfx1100-pool-A)

```bash
# Environment set by workflow
export TEST_COMPONENT=rocblas
export SHARD_INDEX=2
export TOTAL_SHARDS=6
export AMDGPU_FAMILIES=gfx1100

# Execute test script
python test_runner.py
```

---

# Generic Test Runner

**One script for ALL components**

```
Traditional (Bad):              Generic (Good):
───────────────                 ───────────────

if component == "rocblas":      component = env.TEST_COMPONENT
    run_rocblas()               
elif component == "miopen":     # Discover what exists
    run_miopen()                labels = ctest --print-labels
elif ...                        
                                # Filter automatically
100 components                  ctest -L category -L gpu
= 100 if/else blocks            
                                Works for ANY component!
```

---

# How Generic Runner Works

```
Step 1: Discover labels
  $ ctest --print-labels
  → [standard, ex_gpu_gfx1100, quick, ...]

Step 2: Match GPU architecture
  Current: gfx1100
  Available: [gfx1100, gfx110X, gfx11X]
  Match: gfx1100 (exact!)

Step 3: Build filter
  ctest -L standard -L ex_gpu_gfx1100

Step 4: Shard tests
  --tests-information 2,6  (shard 2 of 6)

Step 5: Execute
```

---

# Parallel Execution

```
Without Sharding:               With Sharding (6 shards):
────────────────               ─────────────────────────

┌──────────────┐               ┌────┐ ┌────┐ ┌────┐
│ All 10,000   │               │1666│ │1666│ │1666│
│ tests        │               │test│ │test│ │test│
│              │               └────┘ └────┘ └────┘
│ 4 hours      │               ┌────┐ ┌────┐ ┌────┐
└──────────────┘               │1666│ │1666│ │1666│
                               │test│ │test│ │test│
                               └────┘ └────┘ └────┘

                               All run in parallel
                               40 minutes total
                               6x speedup!
```

---

# Framework Agnostic

**Any test framework works:**

```python
# CTest-based
"rocblas": {
    "test_script": "python test_runner.py",
},

# Pytest-based
"rccl": {
    "test_script": "pytest test_rccl.py -v",
},

# Custom script
"custom": {
    "test_script": "bash run_tests.sh",
},
```

**Workflow doesn't care** - just runs `test_script` and checks exit code!

---

# Load Balancing

**Weighted distribution across runner pools:**

```python
"test-runs-on-labels": [
    {"label": "pool-A", "weight": 0.70},  # 70% of jobs
    {"label": "pool-B", "weight": 0.30},  # 30% of jobs
]
```

**100 jobs →**
- ~70 jobs to pool-A
- ~30 jobs to pool-B

Infrastructure team adjusts weights, no test changes needed!

---

# Adding a New Component

## Step 1: Write tests with labels

```cmake
add_test(NAME my_test COMMAND my_test_exe)
set_tests_properties(my_test PROPERTIES
    LABELS "standard;ex_gpu_gfx1100"
)
```

## Step 2: Add to configuration

```python
test_matrix = {
    "my-component": {
        "job_name": "my-component",
        "total_shards_dict": {"linux": 2},
        "test_script": "python test_runner.py",
    },
}
```

## That's it! Generic runner handles the rest.

---

# Infrastructure Scales Independently

**Infra team adds 10 new machines:**

```python
# ONLY THIS FILE CHANGES:
# amdgpu_family_matrix.py

"gfx1100": {
    "test-runs-on-labels": [
        {"label": "pool-A", "weight": 0.50},  # Was 0.70
        {"label": "pool-B", "weight": 0.30},  # Same
        {"label": "pool-C", "weight": 0.20},  # NEW!
    ]
}

# fetch_test_configurations.py
# NO CHANGES NEEDED!
```

**Jobs automatically distribute 50/30/20 across pools**

---

# Separation of Concerns

```
┌─────────────────────────────────────────────────┐
│ Test Engineers:                                  │
│ • Add components                                 │
│ • Configure sharding                             │
│ • Define test scripts                            │
│ • DON'T manage infrastructure                    │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ Infrastructure Team:                             │
│ • Provision runners                              │
│ • Adjust load balancing                          │
│ • Scale capacity                                 │
│ • DON'T touch test configs                       │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ GitHub Actions:                                  │
│ • Creates matrix jobs                            │
│ • Dispatches to runners                          │
│ • Handles retries                                │
│ • AUTOMATIC - no human intervention              │
└─────────────────────────────────────────────────┘
```

---

# Real Example: rocBLAS

**Configuration:**
```python
"rocblas": {
    "timeout_minutes": 288,
    "total_shards_dict": {"linux": 6},
}
```

**Result:**
- 6 parallel jobs created
- Each job: ~10,000 tests / 6 = ~1,666 tests
- Each job: ~4 hours / 6 = ~40 minutes
- Total wall time: 40 minutes (vs 4 hours sequential)

**6x speedup from sharding!**

---

# Key Benefits

✅ **Configuration-Driven**
   - `test_matrix` is pure data, not code
   - Easy to modify and validate

✅ **Generic & Extensible**
   - One runner for all components
   - Add component = add config entry

✅ **Automatic Parallelism**
   - `total_shards: 6` → 6 parallel jobs
   - Linear speedup with more shards

---

# Key Benefits (continued)

✅ **Framework Agnostic**
   - CTest, pytest, custom - all work
   - Just implement script that reads env vars

✅ **Independent Infrastructure**
   - Infra scales without touching tests
   - Test changes don't affect infrastructure
   - Labels connect them automatically

✅ **Zero Manual Dispatch**
   - No job assignment code
   - GitHub handles scheduling
   - Load balancing automatic

---

# Developer Experience

**Old way:**
1. Write test
2. Figure out which runner
3. Write job dispatch logic
4. Configure workflow
5. Wait for infra provisioning
6. Debug allocation issues
7. Finally run test

**New way:**
1. Write test
2. Add label: `"standard;ex_gpu_gfx1100"`
3. Done!

**Time to first test run:** Minutes (vs days)

---

# Summary

## Three Simple Concepts

1. **Configure tests once** in `test_matrix`
2. **Label tests** with categories and GPUs
3. **GitHub handles the rest**

**No job dispatch code.**
**No runner management.**
**Just configuration and labels.**

---

# Files Reference

| File | Purpose | Runs On |
|------|---------|---------|
| `fetch_test_configurations.py` | Test config (WHAT) | ubuntu-24.04 |
| `amdgpu_family_matrix.py` | Runner pools (WHERE) | ubuntu-24.04 |
| `test_runner.py` | Generic executor | Your GPU runners |
| `test_component.yml` | Matrix workflow | Your GPU runners |

---

# Questions?

**Documentation:** `docs/testing/README.md`

**Key concepts:**
- Configuration-driven (not code-driven)
- Generic test runner (label-based discovery)
- Automatic job dispatch (GitHub + labels)
- Independent scaling (infra + tests separate)
- Framework agnostic (any test framework works)

---

# Thank You!

## TheRock Test Harness
### Simple. Scalable. Framework-Agnostic.
