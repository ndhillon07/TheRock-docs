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

## Framework Agnostic: Plug and Play

The harness is **framework agnostic** - it just runs whatever `test_script` you specify. Use CTest, pytest, custom scripts, anything.

```
┌─────────────────────────────────────────────────────────────┐
│                  Workflow (Same for All)                     │
│                                                              │
│  1. Set environment variables                                │
│  2. Run: ${{ test_script }}                                 │
│  3. Check exit code                                          │
└─────────────────────────────────────────────────────────────┘
                           │
                           │ test_script can be anything
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
   ┌─────────┐        ┌─────────┐      ┌──────────┐
   │ CTest   │        │ pytest  │      │  Custom  │
   │ runner  │        │         │      │  script  │
   └─────────┘        └─────────┘      └──────────┘
        │                  │                  │
        │                  │                  │
   Your tests        Your tests         Your tests
   Any framework!    Any framework!     Any framework!
```

### Different Test Frameworks

```python
test_matrix = {
    # CTest-based (generic runner)
    "rocblas": {
        "test_script": "python .../test_runner.py",  # Uses CTest
    },
    
    # Pytest-based
    "rccl": {
        "test_script": "pytest .../test_rccl.py -v -s",  # Uses pytest
    },
    
    # Custom script
    "rocfft": {
        "test_script": "python .../test_rocfft.py",  # Custom logic
    },
    
    # Shell script
    "custom": {
        "test_script": "bash .../run_custom_tests.sh",  # Bash script
    },
}
```

**The workflow doesn't care** - it just executes `test_script` and checks the exit code.

### Per-Component Custom Settings

Each component can have unique configuration:

```python
test_matrix = {
    "rocblas": {
        "test_script": "python .../test_runner.py",
        "timeout_minutes": 288,
        "total_shards_dict": {"linux": 6},
        
        # Custom container for this component only
        "container_image": "ghcr.io/rocm/rocblas_image@sha256:...",
        
        # Additional container options
        "container_options": ["--cap-add=SYS_PTRACE"],
        
        # Python requirements for this test
        "additional_requirements_files": [
            "share/rocblas/requirements.txt",
        ],
        
        # Exclude specific GPUs
        "exclude_family": {
            "linux": ["gfx1150", "gfx1151"],
        },
        
        # Require multi-GPU
        "multi_gpu": {"linux": ["gfx94X-dcgpu"]},
    },
}
```

### Per-Component Environment & Paths

**File**: `test_runner.py` (for CTest-based tests)

```python
COMPONENT_OVERRIDES = {
    "rocprofiler-compute": {
        # Custom test directory
        "test_dir": ["libexec", "rocprofiler-compute"],
        
        # Additional environment paths (relative to ROCM_PATH)
        "additional_env_paths": {
            "PATH": [["bin"]],
            "LD_LIBRARY_PATH": [["lib"], ["lib", "rocm_sysdeps", "lib"]],
        },
    },
    
    "rocwmma": {
        # Different test directory based on TEST_TYPE
        "test_dir_by_type": {
            "quick": ["bin", "rocwmma", "regression"],
            "standard": ["bin", "rocwmma"],
        },
    },
    
    "rocroller": {
        # Paths from build tree (not install tree)
        "env_prepend_from_therock": {
            "LD_LIBRARY_PATH": [
                ["build", "math-libs", "BLAS", "rocRoller", "dist", "lib"],
            ],
        },
    },
}
```

### Custom Test Scripts

Write your own test script for complete control:

**Example**: `test_executable_scripts/test_my_component.py`

```python
#!/usr/bin/env python3
import os
import subprocess

# Read environment (set by workflow)
component = os.getenv("TEST_COMPONENT")
shard_index = int(os.getenv("SHARD_INDEX", 1))
total_shards = int(os.getenv("TOTAL_SHARDS", 1))
test_type = os.getenv("TEST_TYPE", "standard")

# Custom logic for your component
if test_type == "quick":
    tests = ["smoke_test"]
elif test_type == "standard":
    tests = ["unit_tests", "integration_tests"]
else:
    tests = ["all_tests"]

# Shard tests your way
my_shard = tests[shard_index - 1::total_shards]

# Run with your framework
for test in my_shard:
    result = subprocess.run(
        ["my_test_runner", "--test", test],
        check=False
    )
    if result.returncode != 0:
        sys.exit(1)

print(f"All tests passed for shard {shard_index}/{total_shards}")
```

**Use it**:

```python
test_matrix = {
    "my-component": {
        "test_script": "python .../test_my_component.py",
        "total_shards_dict": {"linux": 4},
    },
}
```

### Environment Variables Available to Scripts

All test scripts receive:

```bash
# Component info
TEST_COMPONENT=rocblas          # Which component
TEST_TYPE=standard              # Test category

# Sharding
SHARD_INDEX=2                   # Current shard (1-indexed)
TOTAL_SHARDS=6                  # Total shards

# GPU info
AMDGPU_FAMILIES=gfx1100        # Target GPU

# Paths
THEROCK_BIN_DIR=./build/dist/rocm/bin
ROCM_PATH=./build/dist/rocm

# GTest (if applicable)
GTEST_SHARD_INDEX=1            # 0-indexed
GTEST_TOTAL_SHARDS=6
```

### Plug and Play: Adding New Framework

**Step 1**: Write test script for your framework

```python
# test_executable_scripts/test_with_my_framework.py
import my_test_framework

def main():
    component = os.getenv("TEST_COMPONENT")
    shard = int(os.getenv("SHARD_INDEX"))
    total = int(os.getenv("TOTAL_SHARDS"))
    
    # Use your framework
    runner = my_test_framework.Runner(component)
    runner.shard(shard, total)
    runner.run()

if __name__ == "__main__":
    main()
```

**Step 2**: Add to test_matrix

```python
test_matrix = {
    "my-component": {
        "test_script": "python .../test_with_my_framework.py",
        "total_shards_dict": {"linux": 4},
    },
}
```

**Done!** The workflow doesn't care what framework you use.

### Real Examples from TheRock

```python
# CTest-based (generic)
"rocblas": {
    "test_script": f"python {_get_script_path('test_runner.py')}",
},

# Pytest-based
"rccl": {
    "test_script": f"pytest {_get_script_path('test_rccl.py')} -v -s",
},

# Custom Python script
"amdsmi": {
    "test_script": f"python {_get_script_path('test_amdsmi.py')}",
},

# Custom with special handling
"hipfft": {
    "test_script": f"python {_get_script_path('test_hipfft.py')}",
    "additional_requirements_files": ["share/hipfft/requirements.txt"],
},
```

### Why This Works

**Workflow is simple**:

```yaml
- name: Test
  run: ${{ fromJSON(inputs.component).test_script }}
```

That's it! The workflow:
- Sets environment variables
- Runs your script
- Checks exit code (0 = pass, non-zero = fail)

**Your script**:
- Reads environment variables
- Runs tests however you want
- Returns exit code

**Result**: Any test framework works!

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

**Four Simple Concepts:**

1. **Configure tests once** in `test_matrix` (any test framework)
2. **Write test script** (CTest, pytest, custom - anything)
3. **Label runner pools** (infrastructure manages capacity)
4. **GitHub handles the rest** (jobs, dispatch, parallelism)

**Plug and Play:**
- Any test framework works (CTest, pytest, custom)
- Any custom settings per component
- Any environment configuration
- Just set `test_script` and environment variables

No job dispatch code. No runner management. Just configuration.
