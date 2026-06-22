# Test Harness: Configuration-Based Generic Testing

A simple, configuration-driven test harness where you define tests once and GitHub Actions handles the rest.

---

## Core Concept

```
┌─────────────────────────────────────────────────────────────┐
│  You Write                GitHub Actions Does               │
│  ─────────                ──────────────────                │
│  • Configuration          • Creates parallel jobs           │
│  • Test labels            • Dispatches to runners           │
│                           • Collects results                │
│                                                             │
│  No job dispatch code needed!                               │
└─────────────────────────────────────────────────────────────┘
```

---

## How It Works

### 1. Define Configuration (Once)

**File**: `build_tools/github_actions/fetch_test_configurations.py`

```python
test_matrix = {
    "rocblas": {
        "job_name": "rocblas",
        "timeout_minutes": 288,
        "test_script": "python .../test_runner.py",
        "platform": ["linux", "windows"],
        "total_shards_dict": {
            "linux": 6,      # ← Creates 6 parallel jobs
            "windows": 6,
        },
    },
    "miopen": {
        "job_name": "miopen",
        "timeout_minutes": 120,
        "test_script": "python .../test_runner.py",
        "platform": ["linux"],
        "total_shards_dict": {
            "linux": 4,      # ← Creates 4 parallel jobs
        },
    },
}
```

### 2. Generic Runner Executes

**File**: `build_tools/github_actions/test_executable_scripts/test_runner.py`

One runner works for ALL components. It:
- Reads `TEST_COMPONENT` env var
- Discovers test labels via `ctest --print-labels`
- Filters by category (`quick`, `standard`, etc.)
- Filters by GPU (`ex_gpu_gfx1100`, etc.)
- Runs tests with automatic sharding

### 3. GitHub Creates Matrix Jobs

```
Configuration:                    GitHub Creates:
─────────────                     ───────────────
rocblas:                          • rocblas (shard 1/6)
  total_shards: 6        ──────>  • rocblas (shard 2/6)
                                  • rocblas (shard 3/6)
                                  • rocblas (shard 4/6)
                                  • rocblas (shard 5/6)
                                  • rocblas (shard 6/6)

miopen:                           • miopen (shard 1/4)
  total_shards: 4        ──────>  • miopen (shard 2/4)
                                  • miopen (shard 3/4)
                                  • miopen (shard 4/4)

Total: 10 parallel jobs (automatic!)
```

### 4. Runner Pools & Dispatch

**File**: `tests/amdgpu_family_matrix.py`

```python
"gfx1100": {
    "linux": {
        "test-runs-on-labels": [
            {"label": "gfx1100-pool-A", "weight": 70},  # 70% of jobs
            {"label": "gfx1100-pool-B", "weight": 30},  # 30% of jobs
        ]
    }
}
```

**Infrastructure team provisions runners**:
```bash
# Pool A: 12 runners
./config.sh --labels linux,gpu,gfx1100-pool-A

# Pool B: 5 runners
./config.sh --labels linux,gpu,gfx1100-pool-B
```

**GitHub automatically dispatches**:
- Job needs: `runs-on: gfx1100-pool-A`
- GitHub finds idle runner with that label
- Job executes
- No manual dispatch needed

---

## Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│ STEP 1: Configuration (fetch_test_configurations.py)                │
└─────────────────────────────────────────────────────────────────────┘

test_matrix = {
    "rocblas": {"total_shards_dict": {"linux": 6}},
    "miopen":  {"total_shards_dict": {"linux": 4}},
}

                    │
                    │ Script generates JSON
                    ▼

┌─────────────────────────────────────────────────────────────────────┐
│ STEP 2: Matrix Generation                                           │
└─────────────────────────────────────────────────────────────────────┘

Output:
{
  "components": [
    {
      "job_name": "rocblas",
      "shard_arr": [1, 2, 3, 4, 5, 6],  ← Array for matrix
      "test_runner": "gfx1100-pool-A"
    },
    {
      "job_name": "miopen",
      "shard_arr": [1, 2, 3, 4],
      "test_runner": "gfx1100-pool-B"
    }
  ]
}

                    │
                    │ GitHub Actions reads JSON
                    ▼

┌─────────────────────────────────────────────────────────────────────┐
│ STEP 3: GitHub Actions Matrix Expansion                             │
└─────────────────────────────────────────────────────────────────────┘

strategy:
  matrix:
    shard: ${{ fromJSON(inputs.component).shard_arr }}

Creates 10 jobs:
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ rocblas 1/6  │  │ rocblas 2/6  │  │ rocblas 3/6  │
  │ runs-on:     │  │ runs-on:     │  │ runs-on:     │
  │ pool-A       │  │ pool-A       │  │ pool-A       │
  └──────────────┘  └──────────────┘  └──────────────┘
  ... (rocblas 4/6, 5/6, 6/6)

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ miopen 1/4   │  │ miopen 2/4   │  │ miopen 3/4   │
  │ runs-on:     │  │ runs-on:     │  │ runs-on:     │
  │ pool-B       │  │ pool-B       │  │ pool-B       │
  └──────────────┘  └──────────────┘  └──────────────┘
  ... (miopen 4/4)

                    │
                    │ GitHub scheduler
                    ▼

┌─────────────────────────────────────────────────────────────────────┐
│ STEP 4: Job Dispatch (Automatic by GitHub)                          │
└─────────────────────────────────────────────────────────────────────┘

Runner Infrastructure:

┌────────────────────────────────────┐
│ Pool A (label: gfx1100-pool-A)     │
│ ┌────────┐ ┌────────┐ ┌────────┐  │
│ │Runner 1│ │Runner 2│ │Runner 3│  │
│ └────────┘ └────────┘ └────────┘  │
│ ... (12 total runners)             │
└────────────────────────────────────┘
      ▲
      │ GitHub matches: "runs-on: gfx1100-pool-A"
      │

┌────────────────────────────────────┐
│ Pool B (label: gfx1100-pool-B)     │
│ ┌────────┐ ┌────────┐             │
│ │Runner 1│ │Runner 2│             │
│ └────────┘ └────────┘             │
│ ... (5 total runners)              │
└────────────────────────────────────┘
      ▲
      │ GitHub matches: "runs-on: gfx1100-pool-B"
      │

                    │
                    │ Jobs execute
                    ▼

┌─────────────────────────────────────────────────────────────────────┐
│ STEP 5: Test Execution (test_runner.py)                             │
└─────────────────────────────────────────────────────────────────────┘

Each job runs:
  export TEST_COMPONENT=rocblas
  export SHARD_INDEX=1
  export TOTAL_SHARDS=6
  export AMDGPU_FAMILIES=gfx1100
  
  python test_runner.py
  
  → Discovers labels: [standard, ex_gpu_gfx1100, ...]
  → Builds filter: ctest -L standard -L ex_gpu_gfx1100
  → Shards tests: --tests-information 1,6
  → Runs tests
  → Reports results
```

---

## Adding a New Component

### Step 1: Write Tests with Labels

**In your component's CMakeLists.txt**:

```cmake
add_test(NAME my_test COMMAND my_test_exe)
set_tests_properties(my_test PROPERTIES
    LABELS "standard;ex_gpu_gfx1100"
)
```

**Label conventions**:
- Category: `quick`, `standard`, `comprehensive`, `full`
- GPU: `ex_gpu_gfx1100`, `ex_gpu_gfx11X` (wildcard), etc.

### Step 2: Add to Configuration

**In `fetch_test_configurations.py`**:

```python
test_matrix = {
    # ... existing components ...
    
    "my-component": {
        "job_name": "my-component",
        "fetch_artifact_args": "--my-component --tests",
        "timeout_minutes": 60,
        "test_script": f"python {_get_script_path('test_runner.py')}",
        "platform": ["linux", "windows"],
        "total_shards_dict": {
            "linux": 2,    # 2 parallel jobs on Linux
            "windows": 1,   # 1 job on Windows
        },
    },
}
```

**That's it!** Generic runner handles the rest.

### Step 3: (Optional) Map Directory Name

**If job name ≠ directory name, in `test_runner.py`**:

```python
COMPONENT_DIR_MAPPING = {
    "my-component": "MyComponentDir",
}
```

---

## Runner Pool Management

### Infrastructure Team Creates Pools

```bash
# Create runners with labels
./config.sh \
  --url https://github.com/ROCm/TheRock \
  --labels linux,gpu,gfx1100-pool-A \
  --name runner-01

./config.sh \
  --url https://github.com/ROCm/TheRock \
  --labels linux,gpu,gfx1100-pool-B \
  --name runner-02
```

### Configure Load Balancing

**In `tests/amdgpu_family_matrix.py`**:

```python
"gfx1100": {
    "linux": {
        "test-runs-on-labels": [
            {"label": "gfx1100-pool-A", "weight": 70},
            {"label": "gfx1100-pool-B", "weight": 30},
        ]
    }
}
```

- 70% of jobs → Pool A
- 30% of jobs → Pool B
- Adjust weights to balance load
- Add/remove pools anytime

### GitHub Handles Dispatch

```
Job Request:                Runner Selection:
────────────                ─────────────────
runs-on: gfx1100-pool-A  →  GitHub finds idle runner
                            with label "gfx1100-pool-A"
                         →  Assigns job to runner
                         →  Runner executes
                         →  Runner becomes idle again
```

**No manual dispatch code. No job assignment logic. Automatic.**

---

## Real Example: rocBLAS

### Configuration

```python
test_matrix = {
    "rocblas": {
        "job_name": "rocblas",
        "fetch_artifact_args": "--blas --tests",
        "timeout_minutes": 288,
        "test_script": f"python {_get_script_path('test_runner.py')}",
        "platform": ["linux", "windows"],
        "total_shards_dict": {
            "linux": 6,
            "windows": 6,
        },
    },
}
```

### Environment

```bash
PROJECTS_TO_TEST="rocblas"
AMDGPU_FAMILIES="gfx1100"
TEST_TYPE="standard"
```

### Generated Matrix

```json
{
  "job_name": "rocblas",
  "total_shards": 6,
  "shard_arr": [1, 2, 3, 4, 5, 6],
  "test_runner": "gfx1100-pool-A"
}
```

### GitHub Creates Jobs

```yaml
# Job 1
name: Test rocblas (shard 1/6) (gfx1100)
runs-on: gfx1100-pool-A
env:
  TEST_COMPONENT: rocblas
  SHARD_INDEX: 1
  TOTAL_SHARDS: 6

# Job 2
name: Test rocblas (shard 2/6) (gfx1100)
runs-on: gfx1100-pool-A
env:
  TEST_COMPONENT: rocblas
  SHARD_INDEX: 2
  TOTAL_SHARDS: 6

# ... jobs 3, 4, 5, 6
```

### Execution

Each job:
1. Downloads artifacts
2. Runs `test_runner.py`
3. Discovers labels: `[standard, ex_gpu_gfx1100, ...]`
4. Filters: `ctest -L standard -L ex_gpu_gfx1100 --tests-information N,6`
5. Tests auto-shard by index
6. Reports results

**Result**: 6x speedup from parallel execution!

---

## Key Configuration Fields

### total_shards_dict

Controls parallelism:

```python
"total_shards_dict": {
    "linux": 6,    # More shards = faster (needs more runners)
    "windows": 2,  # Fewer shards = slower (fewer runners needed)
}
```

### platform

Which OS to run on:

```python
"platform": ["linux", "windows"]  # Both
"platform": ["linux"]             # Linux only
```

### timeout_minutes

Per-job timeout:

```python
"timeout_minutes": 288,  # 4.8 hours per shard
```

### exclude_family

Skip specific GPUs:

```python
"exclude_family": {
    "linux": ["gfx1150", "gfx1151"],
}
```

### multi_gpu

Require multi-GPU runners:

```python
"multi_gpu": {
    "linux": ["gfx94X-dcgpu", "gfx950-dcgpu"],
}
```

---

## Quick Reference

### Add component
1. Add test labels in CMake
2. Add entry to `test_matrix`
3. Done!

### Adjust parallelism
1. Change `total_shards_dict` value
2. Higher = more parallel jobs
3. Done!

### Add runner pool
1. Infrastructure provisions runners with labels
2. Add to `test-runs-on-labels` with weight
3. GitHub auto-dispatches
4. Done!

### Debug
```bash
# See what tests would run
export TEST_COMPONENT=rocblas
export AMDGPU_FAMILIES=gfx1100
cd build/dist/rocm/bin/rocblas
ctest -L standard -L ex_gpu_gfx1100 -N

# See available labels
ctest --print-labels
```

---

## Why This Works

### ✓ Configuration-Driven
- `test_matrix` is pure data
- No procedural job creation
- Easy to modify

### ✓ Generic Runner
- One runner for all components
- Label-based filtering
- Automatic discovery

### ✓ Automatic Parallelism
- `total_shards: 6` → 6 jobs
- Linear speedup
- No sharding code

### ✓ Independent Infrastructure
- Test authors: Add tests + labels
- Infra teams: Manage runner pools
- GitHub: Handles dispatch
- No coordination needed

### ✓ Zero Manual Dispatch
- No job assignment code
- No runner selection logic
- Labels + GitHub = automatic

---

## Files Reference

| File | Purpose |
|------|---------|
| `build_tools/github_actions/fetch_test_configurations.py` | Configuration → Matrix JSON |
| `build_tools/github_actions/test_executable_scripts/test_runner.py` | Generic test executor |
| `.github/workflows/test_component.yml` | Reusable matrix workflow |
| `tests/amdgpu_family_matrix.py` | Runner pool configuration |

---

## Summary

**Three Simple Concepts:**

1. **Configure tests once** in `test_matrix`
2. **Label tests** with categories and GPUs
3. **GitHub handles the rest** (jobs, dispatch, parallelism)

No job dispatch code. No runner management. Just configuration and labels.
