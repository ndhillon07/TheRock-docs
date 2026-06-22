# Test Harness: Configuration-Driven Testing

Configuration defines tests. Labels define runners. GitHub dispatches automatically.

---

## Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CONFIGURATION FILES                              │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────┐    ┌─────────────────────────────────┐
│ fetch_test_configurations.py     │    │ amdgpu_family_matrix.py         │
│ ─────────────────────────────    │    │ ───────────────────────         │
│ test_matrix = {                  │    │ "gfx1100": {                    │
│   "rocblas": {                   │    │   "test-runs-on-labels": [      │
│     "total_shards": 6,           │    │     {"label":                   │
│     "timeout": 288,              │    │      "linux-gfx1100-pool-A",    │
│     "test_script": "...",        │    │      "weight": 0.70},           │
│   },                             │    │     {"label":                   │
│   "miopen": {                    │    │      "linux-gfx1100-pool-B",    │
│     "total_shards": 4,           │    │      "weight": 0.30}            │
│     "timeout": 120,              │    │   ]                             │
│     "test_script": "...",        │    │ }                               │
│   }                              │    │                                 │
│ }                                │    │ Defines: WHERE to run           │
│                                  │    │   • Runner labels               │
│ Defines: WHAT to test            │    │   • Load balancing              │
│   • Multiple components          │    │   • GPU architectures           │
│   • Different parallelism        │    │                                 │
│   • Different test scripts       │    │                                 │
└──────────────┬───────────────────┘    └──────────────┬──────────────────┘
               │                                       │
               │ Generates JSON                        │ Provides labels + weights
               │                                       │
               │ Script uses weights to select label:  │
               │ • 70% chance → pool-A                 │
               │ • 30% chance → pool-B                 │
               ▼                                       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMBINED CONFIGURATION (JSON)                         │
│ ──────────────────────────────────────────────────────────────          │
│ {                                                                        │
│   "components": [                                                        │
│     {                                                                    │
│       "job_name": "rocblas",                                             │
│       "shard_arr": [1, 2, 3, 4, 5, 6],        ← 6 parallel jobs         │
│       "test_runner": "linux-gfx1100-pool-A",  ← Selected via 70% weight │
│       "timeout_minutes": 288,                                            │
│       "test_script": "python test_runner.py"                             │
│     },                                                                   │
│     {                                                                    │
│       "job_name": "miopen",                                              │
│       "shard_arr": [1, 2, 3, 4],              ← 4 parallel jobs         │
│       "test_runner": "linux-gfx1100-pool-B",  ← Selected via 30% weight │
│       "timeout_minutes": 120,                                            │
│       "test_script": "pytest test_miopen.py"                             │
│     }                                                                    │
│   ]                                                                      │
│ }                                                                        │
│                                                                          │
│ Label selection done by config script (not GitHub)                       │
│ Total: 10 parallel jobs (6 rocblas + 4 miopen)                          │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               │ Fed to GitHub Actions
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     GITHUB ACTIONS MATRIX EXPANSION                      │
│ ───────────────────────────────────────────────────────────────         │
│ GitHub creates 10 parallel jobs from 2 components:                       │
│                                                                          │
│ rocblas (6 shards) → All use pool-A:                                    │
│ ┌───────────────────────────┐ ┌───────────────────────────┐            │
│ │ Job: rocblas 1/6          │ │ Job: rocblas 2/6          │            │
│ │ runs-on:                  │ │ runs-on:                  │            │
│ │ linux-gfx1100-pool-A      │ │ linux-gfx1100-pool-A      │            │
│ └───────────────────────────┘ └───────────────────────────┘            │
│ ... (rocblas 3/6, 4/6, 5/6, 6/6 all use pool-A)                         │
│                                                                          │
│ miopen (4 shards) → All use pool-B:                                     │
│ ┌───────────────────────────┐ ┌───────────────────────────┐            │
│ │ Job: miopen 1/4           │ │ Job: miopen 2/4           │            │
│ │ runs-on:                  │ │ runs-on:                  │            │
│ │ linux-gfx1100-pool-B      │ │ linux-gfx1100-pool-B      │            │
│ └───────────────────────────┘ └───────────────────────────┘            │
│ ... (miopen 3/4, 4/4 use pool-B)                                        │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               │ 6 jobs → pool-A, 4 jobs → pool-B
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    GITHUB SCHEDULER (AUTOMATIC)                          │
│ ───────────────────────────────────────────────────────────────         │
│ Matches jobs to runners based on exact "runs-on" label                  │
│ (GitHub doesn't do load balancing - just matches label to runner)       │
│                                                                          │
│ Job Queue A (6 rocblas jobs):    Job Queue B (4 miopen jobs):           │
│ ┌───────────────────────────┐    ┌───────────────────────────┐         │
│ │ • rocblas 1/6 → Running   │    │ • miopen 1/4 → Running    │         │
│ │ • rocblas 2/6 → Running   │    │ • miopen 2/4 → Running    │         │
│ │ • rocblas 3/6 → Queued    │    │ • miopen 3/4 → Queued     │         │
│ │ • rocblas 4/6 → Queued    │    │ • miopen 4/4 → Queued     │         │
│ │ • rocblas 5/6 → Queued    │    └───────────────────────────┘         │
│ │ • rocblas 6/6 → Queued    │                                           │
│ └───────────────────────────┘                                           │
│         ↓                                  ↓                             │
│   Matched to pool-A              Matched to pool-B                       │
│         ↓                                  ↓                             │
│ ┌──────────────────────────┐    ┌──────────────────────────┐           │
│ │ linux-gfx1100-pool-A:    │    │ linux-gfx1100-pool-B:    │           │
│ │   Runner1: [rocblas 1/6] │    │   Runner1: [miopen 1/4]  │           │
│ │      → Finishes          │    │      → Finishes          │           │
│ │      → Takes rocblas 3/6 │    │      → Takes miopen 3/4  │           │
│ │   Runner2: [rocblas 2/6] │    │   Runner2: [miopen 2/4]  │           │
│ │      → Finishes          │    │      → Finishes          │           │
│ │      → Takes rocblas 4/6 │    │      → Takes miopen 4/4  │           │
│ │   Runner3: [Idle]        │    └──────────────────────────┘           │
│ └──────────────────────────┘                                            │
│                                                                          │
│ Key Points:                                                              │
│ • Config script selected labels (pool-A for rocblas, pool-B for miopen) │
│ • GitHub just matches: jobs needing pool-A → pool-A runners             │
│ •                      jobs needing pool-B → pool-B runners             │
│ • Runners pick up next job from THEIR pool's queue                      │
│ • NO cross-pool assignment (pool-A runners don't take pool-B jobs)     │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               │ Jobs executing on runners
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    TEST EXECUTION (Your GPU Runners)                     │
│ ───────────────────────────────────────────────────────────────         │
│ Runner: linux-gfx1100-pool-A-runner-03                                  │
│                                                                          │
│ $ export TEST_COMPONENT=rocblas                                          │
│ $ export SHARD_INDEX=2                                                   │
│ $ export TOTAL_SHARDS=6                                                  │
│ $ export AMDGPU_FAMILIES=gfx1100                                        │
│ $ python test_runner.py                                                  │
│                                                                          │
│ [Test execution happens on GPU hardware]                                 │
│ → Exit code 0 = Pass                                                     │
│ → Exit code non-zero = Fail                                              │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key Points:**
1. **Config** decides: components, shards, scripts
2. **AMDGPU matrix** provides: runner labels (runs-on)
3. **Workflow** parallelizes: creates N jobs from shard array
4. **GitHub** schedules: matches jobs to labeled runners automatically

---

## Core Concept

```
Configuration (JSON)  +  Labels (Infrastructure)  =  Automatic Dispatch
     ↓                        ↓                            ↓
  What to test           Where to run                 GitHub handles
  How many shards        Runner pools                 job scheduling
  What script            GPU architectures            load balancing
```

---

## The Two Configurations

### 1. Test Configuration (WHAT to test)

**File**: `build_tools/github_actions/fetch_test_configurations.py`

```python
test_matrix = {
    "rocblas": {
        "total_shards_dict": {"linux": 6},      # 6 parallel jobs
        "timeout_minutes": 288,
        "test_script": "python test_runner.py", # Any script works
    },
    "miopen": {
        "total_shards_dict": {"linux": 4},      # 4 parallel jobs
        "timeout_minutes": 120,
        "test_script": "pytest test_miopen.py",
    },
}
```

**Generates JSON** → GitHub uses for matrix expansion

### 2. Runner Configuration (WHERE to run)

**File**: `build_tools/github_actions/amdgpu_family_matrix.py`

```python
"gfx1100": {
    "linux": {
        "test-runs-on-labels": [
            {"label": "linux-gfx1100-pool-A", "weight": 0.70},
            {"label": "linux-gfx1100-pool-B", "weight": 0.30},
        ]
    }
}
```

**Defines**: Runner pools per GPU architecture with load balancing

---

## How It Works

```
Step 1: Configuration Script (ubuntu-24.04)
────────────────────────────────────────────
fetch_test_configurations.py reads test_matrix
  → Generates JSON: {rocblas: {shards: 6, runner: "gfx1100-pool-A"}}
  → Outputs to GitHub Actions

Step 2: GitHub Matrix Expansion
────────────────────────────────
Reads JSON, creates jobs:
  • rocblas shard 1/6 → runs-on: gfx1100-pool-A
  • rocblas shard 2/6 → runs-on: gfx1100-pool-A
  • ... (6 total jobs)

Step 3: GitHub Dispatches to Runners
─────────────────────────────────────
Job needs: "gfx1100-pool-A"
  → GitHub finds idle runner with that label
  → Assigns job to runner
  → Automatic scheduling

Step 4: Test Execution (Your GPU Runners)
──────────────────────────────────────────
Runner executes:
  export TEST_COMPONENT=rocblas
  export SHARD_INDEX=2
  export TOTAL_SHARDS=6
  python test_runner.py    ← Your test script
```

---

## Labels = Infrastructure Components

**Labels are runner pool identifiers**

```python
# Heavy runners for compute-intensive tests
"test-runs-on-labels": [
    {"label": "linux-gfx950-8gpu-heavy", "weight": 1.0}
]

# Light runners for quick tests
"test-runs-on-labels": [
    {"label": "linux-gfx1100-1gpu-light", "weight": 1.0}
]

# Multiple pools for load balancing
"test-runs-on-labels": [
    {"label": "pool-datacenter-A", "weight": 0.60},
    {"label": "pool-datacenter-B", "weight": 0.40},
]
```

**Infrastructure team provisions runners** with these labels → GitHub dispatches automatically

---

## Framework Agnostic: Any Test Script Works

The workflow just executes your `test_script`:

```yaml
# In workflow
- name: Test
  run: ${{ test_script }}  # Whatever you specified
```

**Examples:**

```python
# CTest-based (test_runner.py is starter script)
"rocblas": {
    "test_script": "python test_runner.py",  # Discovers & runs ctest
},

# Pytest-based
"rccl": {
    "test_script": "pytest test_rccl.py -v -s",
},

# Bash script
"custom": {
    "test_script": "bash run_my_tests.sh",
},

# Any executable
"my-test": {
    "test_script": "./my_custom_test_binary --args",
},
```

**All scripts receive environment variables:**
- `TEST_COMPONENT` - Which component
- `SHARD_INDEX` / `TOTAL_SHARDS` - For sharding
- `AMDGPU_FAMILIES` - GPU architecture
- `TEST_TYPE` - quick/standard/comprehensive

**Your script:**
1. Reads env vars
2. Runs tests however you want
3. Returns exit code (0=pass, non-zero=fail)

---

## Scaling Examples

### Scale by Test Needs

**Heavy compute tests** → Beefy runners:

```python
"miopen": {
    "total_shards_dict": {"linux": 4},
    # Infrastructure provides heavy runners for this GPU
}

# In amdgpu_family_matrix.py
"gfx950": {
    "test-runs-on": "linux-gfx950-8gpu-128gb-ram",  # Heavy!
}
```

**Light tests** → Small runners:

```python
"sanity": {
    "total_shards_dict": {"linux": 1},
}

# In amdgpu_family_matrix.py
"gfx1100": {
    "test-runs-on": "linux-gfx1100-1gpu-16gb-ram",  # Light
}
```

### Scale by Architecture

**Different GPU families = Different runners:**

```python
# amdgpu_family_matrix.py
"gfx1100": {
    "linux": {
        "test-runs-on": "linux-gfx1100-pool",
    }
},
"gfx950": {
    "linux": {
        "test-runs-on": "linux-gfx950-pool",
    }
},
"gfx94X-dcgpu": {
    "linux": {
        "test-runs-on-multi-gpu": "linux-gfx942-8gpu-pool",  # Multi-GPU!
    }
}
```

**Environment `AMDGPU_FAMILIES=gfx950`** → Automatically uses gfx950 runners

### Scale by Load Balancing

**Distribute across multiple datacenters:**

```python
"gfx1100": {
    "test-runs-on-labels": [
        {"label": "datacenter-US-east", "weight": 0.50},
        {"label": "datacenter-US-west", "weight": 0.30},
        {"label": "datacenter-EU", "weight": 0.20},
    ]
}
```

50% of jobs → US-east, 30% → US-west, 20% → EU

---

## Independent Scaling

**Test engineers** change `test_matrix`:
- Add components
- Adjust shards (parallelism)
- Change timeouts

**Infrastructure team** changes `amdgpu_family_matrix.py`:
- Add runner pools
- Adjust weights
- Provision new GPUs

**No coordination needed!** They work independently.

**Example**: Infrastructure adds 10 new gfx1100 machines

```python
# ONLY infrastructure file changes
"gfx1100": {
    "test-runs-on-labels": [
        {"label": "pool-A", "weight": 0.40},  # Was 0.70
        {"label": "pool-B", "weight": 0.30},  # Same
        {"label": "pool-C", "weight": 0.30},  # NEW!
    ]
}

# Test configuration - NO CHANGES
# Jobs automatically use new pool
```

---

## Adding a New Component

### Step 1: Configure Test

```python
# In fetch_test_configurations.py
test_matrix = {
    "my-component": {
        "total_shards_dict": {"linux": 2},
        "timeout_minutes": 60,
        "test_script": "python test_my_component.py",
        "platform": ["linux"],
    },
}
```

### Step 2: Write Test Script

```python
#!/usr/bin/env python3
# test_my_component.py
import os, subprocess

component = os.getenv("TEST_COMPONENT")
shard = int(os.getenv("SHARD_INDEX"))
total = int(os.getenv("TOTAL_SHARDS"))

# Your test logic here
result = subprocess.run(["my_test_runner", f"--shard={shard}/{total}"])
exit(result.returncode)
```

### Step 3: Done!

**GitHub automatically:**
- Creates 2 jobs (total_shards=2)
- Dispatches to available runners
- Runs your test script
- Collects results

---

## Configuration Reference

### test_matrix Fields

| Field | Purpose | Example |
|-------|---------|---------|
| `job_name` | Display name | `"rocblas"` |
| `total_shards_dict` | Parallel jobs per platform | `{"linux": 6, "windows": 2}` |
| `timeout_minutes` | Job timeout | `288` |
| `test_script` | What to execute | `"python test_runner.py"` |
| `platform` | Which OS | `["linux", "windows"]` |
| `exclude_family` | Skip GPU families | `{"linux": ["gfx1150"]}` |
| `multi_gpu` | Require multi-GPU | `{"linux": ["gfx94X-dcgpu"]}` |

### amdgpu_family_matrix Fields

| Field | Purpose |
|-------|---------|
| `test-runs-on` | Single runner label |
| `test-runs-on-labels` | Multiple pools with weights |
| `test-runs-on-multi-gpu` | Multi-GPU runner label |
| `family` | GPU family name |

---

## Example: Complete Flow

**Test Configuration:**

```python
test_matrix = {
    "rocblas": {
        "total_shards_dict": {"linux": 6},
        "test_script": "python test_runner.py",
    }
}
```

**Runner Configuration:**

```python
"gfx1100": {
    "linux": {
        "test-runs-on-labels": [
            {"label": "linux-gfx1100-pool-A", "weight": 0.70},
            {"label": "linux-gfx1100-pool-B", "weight": 0.30},
        ]
    }
}
```

**Environment:** `AMDGPU_FAMILIES=gfx1100`

**Result:**

1. **Config script generates:**
   ```json
   {
     "job_name": "rocblas",
     "shard_arr": [1,2,3,4,5,6],
     "test_runner": "linux-gfx1100-pool-A"
   }
   ```

2. **GitHub creates 6 jobs:**
   - rocblas (1/6) → runs-on: linux-gfx1100-pool-A
   - rocblas (2/6) → runs-on: linux-gfx1100-pool-A
   - ... (6 total)

3. **GitHub dispatches:**
   - Finds idle runners with label "linux-gfx1100-pool-A"
   - Assigns jobs to runners
   - 70% use pool-A, 30% use pool-B (per weights)

4. **Each runner executes:**
   ```bash
   python test_runner.py
   ```
   (With SHARD_INDEX=1, SHARD_INDEX=2, etc.)

---

## Key Insight

**Configuration = What you want**
- Components to test
- Parallelism (shards)
- Test scripts

**Labels = Where to run**
- Runner pools
- GPU architectures
- Hardware specs (heavy vs light)

**GitHub = How it happens**
- Creates jobs from config
- Matches jobs to runners via labels
- Handles scheduling, retries, load balancing

**You configure. GitHub orchestrates. Infrastructure scales.**

---

## Files

| File | Purpose | Runs On |
|------|---------|---------|
| `fetch_test_configurations.py` | Generate JSON config | ubuntu-24.04 |
| `amdgpu_family_matrix.py` | Define runner pools | ubuntu-24.04 |
| `test_runner.py` (or your script) | Execute tests | YOUR GPU runners |

---

## Summary

1. **Configure once** in `test_matrix` and `amdgpu_family_matrix.py`
2. **Labels connect** tests to infrastructure
3. **GitHub dispatches** automatically
4. **Scale independently** - tests and infrastructure don't coordinate
5. **Any framework works** - just implement a script

**No job dispatch code. No manual scheduling. Just configuration and labels.**
