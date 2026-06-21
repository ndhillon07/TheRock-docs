# Configuration-Driven Test Matrix

How `fetch_test_configurations.py` generates matrix jobs for parallel test execution.

---

## Overview

TheRock uses a **configuration-driven approach** where a single Python script (`fetch_test_configurations.py`) reads a declarative configuration and generates a JSON matrix that GitHub Actions uses to create parallel jobs automatically.

**Key Insight:** You configure **WHAT** you want (components, shards, platforms), and GitHub Actions handles **HOW** to execute it (job creation, scheduling, parallel execution).

---

## The Configuration Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONFIGURATION-DRIVEN FLOW                             │
└─────────────────────────────────────────────────────────────────────────┘

Step 1: Configuration Definition (test_matrix dict)
────────────────────────────────────────────────────

┌──────────────────────────────────────────────────────────────────┐
│ fetch_test_configurations.py                                     │
│                                                                   │
│ test_matrix = {                                                  │
│     "rocblas": {                                                 │
│         "job_name": "rocblas",                                   │
│         "timeout_minutes": 288,                                  │
│         "total_shards_dict": {                                   │
│             "linux": 6,      ← 6 parallel jobs                  │
│             "windows": 6     ← 6 parallel jobs                  │
│         },                                                       │
│         "test_script": "python .../test_runner.py",             │
│     },                                                           │
│     "miopen": {                                                  │
│         "job_name": "miopen",                                    │
│         "timeout_minutes": 120,                                  │
│         "total_shards_dict": {                                   │
│             "linux": 4,      ← 4 parallel jobs                  │
│             "windows": 4     ← 4 parallel jobs                  │
│         },                                                       │
│         "test_script": "python .../test_runner.py",             │
│     },                                                           │
│     # ... more components ...                                    │
│ }                                                                │
└──────────────────────────────────────────────────────────────────┘

Step 2: Dynamic Matrix Generation (run() function)
───────────────────────────────────────────────────

Input (environment variables):
  PROJECTS_TO_TEST = "rocblas,miopen"
  AMDGPU_FAMILIES = "gfx1100"
  TEST_TYPE = "standard"

Processing:
┌──────────────────────────────────────────────────────────────────┐
│ for component in ["rocblas", "miopen"]:                         │
│     config = test_matrix[component]                              │
│     total_shards = config["total_shards_dict"]["linux"]          │
│                                                                   │
│     # Create shard array: [1, 2, 3, ..., total_shards]          │
│     shard_arr = [1, 2, 3, 4, 5, 6]  # For rocblas (6 shards)    │
│                                                                   │
│     # Add to components list                                     │
│     components.append({                                          │
│         "job_name": "rocblas",                                   │
│         "total_shards": 6,                                       │
│         "shard_arr": [1, 2, 3, 4, 5, 6],                        │
│         "test_script": "python .../test_runner.py",             │
│         "timeout_minutes": 288,                                  │
│         # ... other fields ...                                   │
│     })                                                           │
└──────────────────────────────────────────────────────────────────┘

Output (JSON to GitHub Actions):
┌──────────────────────────────────────────────────────────────────┐
│ {                                                                │
│   "components": [                                                │
│     {                                                            │
│       "job_name": "rocblas",                                     │
│       "total_shards": 6,                                         │
│       "shard_arr": [1, 2, 3, 4, 5, 6],  ← Array triggers matrix │
│       "test_script": "python .../test_runner.py",               │
│       "timeout_minutes": 288                                     │
│     },                                                           │
│     {                                                            │
│       "job_name": "miopen",                                      │
│       "total_shards": 4,                                         │
│       "shard_arr": [1, 2, 3, 4],        ← Array triggers matrix │
│       "test_script": "python .../test_runner.py",               │
│       "timeout_minutes": 120                                     │
│     }                                                            │
│   ]                                                              │
│ }                                                                │
└──────────────────────────────────────────────────────────────────┘

Step 3: GitHub Actions Matrix Expansion
────────────────────────────────────────

Workflow iterates over components array AND shard array:

┌──────────────────────────────────────────────────────────────────┐
│ .github/workflows/test_component.yml                             │
│                                                                   │
│ jobs:                                                            │
│   test_component:                                                │
│     strategy:                                                    │
│       matrix:                                                    │
│         shard: ${{ fromJSON(inputs.component).shard_arr }}       │
│     runs-on: ...                                                 │
│     env:                                                         │
│       SHARD_INDEX: ${{ matrix.shard }}                          │
│       TOTAL_SHARDS: ${{ fromJSON(inputs.component).total_shards }}│
│     steps:                                                       │
│       - name: Test                                               │
│         run: ${{ fromJSON(inputs.component).test_script }}       │
└──────────────────────────────────────────────────────────────────┘

GitHub Actions creates jobs:
  For rocblas (shard_arr = [1,2,3,4,5,6]):
    - Job: test_component (rocblas, shard 1/6)
    - Job: test_component (rocblas, shard 2/6)
    - Job: test_component (rocblas, shard 3/6)
    - Job: test_component (rocblas, shard 4/6)
    - Job: test_component (rocblas, shard 5/6)
    - Job: test_component (rocblas, shard 6/6)

  For miopen (shard_arr = [1,2,3,4]):
    - Job: test_component (miopen, shard 1/4)
    - Job: test_component (miopen, shard 2/4)
    - Job: test_component (miopen, shard 3/4)
    - Job: test_component (miopen, shard 4/4)

Total: 10 parallel jobs created automatically!

Step 4: Parallel Execution
───────────────────────────

All 10 jobs run in parallel (subject to runner availability):

Runner Pool A:                   Runner Pool B:
┌──────────────────┐            ┌──────────────────┐
│ rocblas 1/6      │            │ miopen 1/4       │
│ Running...       │            │ Running...       │
└──────────────────┘            └──────────────────┘

Runner Pool A:                   Runner Pool B:
┌──────────────────┐            ┌──────────────────┐
│ rocblas 2/6      │            │ miopen 2/4       │
│ Running...       │            │ Running...       │
└──────────────────┘            └──────────────────┘

... (more runners executing other shards)

Each job executes independently:
  - Downloads same artifacts
  - Runs test_runner.py with different SHARD_INDEX
  - Tests automatically distributed by shard
  - Results reported separately
```

---

## Configuration Structure Deep Dive

### The test_matrix Dictionary

This is the **single source of truth** for all test configurations:

```python
test_matrix = {
    # Key: component identifier (used in PROJECTS_TO_TEST)
    "component-name": {
        # Display name in GitHub UI
        "job_name": "display-name",
        
        # Artifact download arguments
        "fetch_artifact_args": "--component --tests",
        
        # Job timeout (in minutes)
        "timeout_minutes": 120,
        
        # Test execution script
        "test_script": "python .../test_runner.py",
        
        # Supported platforms
        "platform": ["linux", "windows"],
        
        # Sharding configuration PER PLATFORM
        "total_shards_dict": {
            "linux": 6,     # Creates 6 parallel jobs on Linux
            "windows": 4,   # Creates 4 parallel jobs on Windows
        },
        
        # Optional: Custom container image (Linux only)
        "container_image": "ghcr.io/rocm/custom@sha256:...",
        
        # Optional: Additional container options
        "container_options": ["--cap-add=SYS_PTRACE"],
        
        # Optional: Exclude specific GPU families
        "exclude_family": {
            "linux": ["gfx1150", "gfx1151"],
            "windows": ["gfx1150"],
        },
        
        # Optional: Multi-GPU requirements
        "multi_gpu": {
            "linux": ["gfx94X-dcgpu", "gfx950-dcgpu"],
        },
        
        # Optional: Additional Python requirements
        "additional_requirements_files": [
            "share/component/requirements.txt",
        ],
        
        # Optional: Expected failure (informational run)
        "expect_failure": True,
    },
}
```

### Key Configuration Fields

#### total_shards_dict: The Parallelism Control

**This is how you control parallel execution!**

```python
"total_shards_dict": {
    "linux": 6,    # Creates jobs: component (shard 1/6), (shard 2/6), ..., (shard 6/6)
    "windows": 2,  # Creates jobs: component (shard 1/2), (shard 2/2)
}
```

**Effect:**
- `total_shards: 1` → 1 job (no parallelism)
- `total_shards: 6` → 6 jobs running in parallel
- `total_shards: 12` → 12 jobs running in parallel

**When to use more shards:**
- Long-running test suites (> 1 hour)
- Want faster feedback
- Have available runner capacity

**When to use fewer shards:**
- Short test suites (< 30 min)
- Limited runner capacity
- Tests don't parallelize well

#### shard_arr: The Generated Array

**You DON'T write this!** It's generated automatically:

```python
# From configuration:
"total_shards_dict": {"linux": 6}

# Generated during run():
"shard_arr": [1, 2, 3, 4, 5, 6]

# Used by GitHub Actions matrix:
strategy:
  matrix:
    shard: [1, 2, 3, 4, 5, 6]  # 6 jobs created
```

---

## Real Example: rocBLAS Configuration

### Configuration

```python
test_matrix = {
    "rocblas": {
        "job_name": "rocblas",
        "fetch_artifact_args": "--blas --tests",
        "timeout_minutes": 288,  # ~4.8 hours per shard
        "test_script": f"python {_get_script_path('test_runner.py')}",
        "platform": ["linux", "windows"],
        "total_shards_dict": {
            "linux": 6,
            "windows": 6,
        },
    },
}
```

### Generated Output (for Linux, gfx1100)

```json
{
  "job_name": "rocblas",
  "total_shards": 6,
  "shard_arr": [1, 2, 3, 4, 5, 6],
  "test_script": "python build_tools/github_actions/test_executable_scripts/test_runner.py",
  "timeout_minutes": 288,
  "test_type": "standard",
  "fetch_artifact_args": "--blas --tests",
  "container_options": "--ipc host --user 0:0 ... --device /dev/kfd ...",
  "test_runner": "gfx1100-pool-A"
}
```

### GitHub Actions Expansion

**Workflow receives this config once, creates 6 jobs:**

```yaml
name: Test rocblas (shard 1/6) (gfx1100)
runs-on: gfx1100-pool-A
timeout-minutes: 288
env:
  SHARD_INDEX: 1
  TOTAL_SHARDS: 6
  TEST_TYPE: standard
  TEST_COMPONENT: rocblas
steps:
  - run: python .../test_runner.py

---

name: Test rocblas (shard 2/6) (gfx1100)
runs-on: gfx1100-pool-A
timeout-minutes: 288
env:
  SHARD_INDEX: 2
  TOTAL_SHARDS: 6
  ...

# ... (shards 3, 4, 5, 6)
```

### Parallel Execution

All 6 jobs run simultaneously:

```
Timeline:

0 min    |  All 6 jobs start downloading artifacts
5 min    |  All 6 jobs start running test_runner.py
         |  Each discovers labels, filters tests, runs ctest
         |
         |  Shard 1: Running tests 1, 7, 13, 19, 25, ...
         |  Shard 2: Running tests 2, 8, 14, 20, 26, ...
         |  Shard 3: Running tests 3, 9, 15, 21, 27, ...
         |  Shard 4: Running tests 4, 10, 16, 22, 28, ...
         |  Shard 5: Running tests 5, 11, 17, 23, 29, ...
         |  Shard 6: Running tests 6, 12, 18, 24, 30, ...
         |
288 min  |  All shards complete (or timeout)

Result: 6x speedup compared to running sequentially!
```

---

## How Different Components Get Different Configurations

### Example: Multiple Components in One Workflow

```python
# Configuration
test_matrix = {
    "rocblas": {
        "job_name": "rocblas",
        "timeout_minutes": 288,
        "total_shards_dict": {"linux": 6},  # Heavy parallelism
    },
    "miopen": {
        "job_name": "miopen",
        "timeout_minutes": 120,
        "total_shards_dict": {"linux": 4},  # Medium parallelism
    },
    "rocrand": {
        "job_name": "rocrand",
        "timeout_minutes": 15,
        "total_shards_dict": {"linux": 1},  # No parallelism needed
    },
}

# Environment
PROJECTS_TO_TEST = "rocblas,miopen,rocrand"
AMDGPU_FAMILIES = "gfx1100"
```

### Workflow Calls test_component.yml Multiple Times

The parent workflow iterates over the components array:

```yaml
jobs:
  configure_tests:
    runs-on: ubuntu-latest
    outputs:
      components: ${{ steps.config.outputs.components }}
    steps:
      - run: python fetch_test_configurations.py
        # Outputs: [rocblas_config, miopen_config, rocrand_config]

  test_component:
    needs: configure_tests
    strategy:
      matrix:
        component: ${{ fromJSON(needs.configure_tests.outputs.components) }}
    uses: ./.github/workflows/test_component.yml
    with:
      component: ${{ toJSON(matrix.component) }}
```

### Result: 3 Component Groups, 11 Total Jobs

**Component: rocblas**
- total_shards: 6
- Jobs created: 6 (rocblas 1/6, 2/6, 3/6, 4/6, 5/6, 6/6)

**Component: miopen**
- total_shards: 4
- Jobs created: 4 (miopen 1/4, 2/4, 3/4, 4/4)

**Component: rocrand**
- total_shards: 1
- Jobs created: 1 (rocrand 1/1)

**Total: 11 jobs running in parallel!**

---

## Configuration-Driven Filtering

### By Project

```bash
# Test only specific components
PROJECTS_TO_TEST="rocblas,miopen"

# Test all components
PROJECTS_TO_TEST="*"
```

### By GPU Family

```bash
# Only include tests compatible with gfx1100
AMDGPU_FAMILIES="gfx1100"

# Component config can exclude families:
"exclude_family": {
    "linux": ["gfx1150", "gfx1151"]  # Skip these architectures
}
```

### By Test Type

```bash
# Quick tests: Force total_shards=1 regardless of config
TEST_TYPE="quick"

# Standard tests: Use configured shards
TEST_TYPE="standard"
```

### By Platform

```python
# In test_matrix:
"platform": ["linux", "windows"]  # Run on both

# Or:
"platform": ["linux"]  # Linux only
```

### By Test Labels

```bash
# Only run specific labeled tests
TEST_LABELS='["test:rocblas", "test:miopen"]'
```

---

## Matrix Generation Algorithm

Here's the simplified logic in `fetch_test_configurations.py`:

```python
def run():
    # Read inputs
    projects_to_test = os.getenv("PROJECTS_TO_TEST", "*")
    platform = os.getenv("RUNNER_OS", "linux").lower()
    test_type = os.getenv("TEST_TYPE", "standard")
    
    all_components = []
    
    # Iterate over test_matrix
    for key, config in test_matrix.items():
        # Filter 1: Is this component requested?
        if key not in projects_to_test and "*" not in projects_to_test:
            continue
        
        # Filter 2: Is this platform supported?
        if platform not in config["platform"]:
            continue
        
        # Filter 3: Is this GPU family excluded?
        if is_excluded(key, platform, amdgpu_families):
            continue
        
        # Build component config
        total_shards = config["total_shards_dict"][platform]
        
        # Quick tests always use 1 shard
        if test_type == "quick":
            total_shards = 1
        
        component_config = {
            **config,
            "total_shards": total_shards,
            "shard_arr": list(range(1, total_shards + 1)),
            "test_type": test_type,
        }
        
        all_components.append(component_config)
    
    # Output JSON for GitHub Actions
    print(json.dumps({"components": all_components}))
```

**Key Points:**
1. **Declarative**: test_matrix is pure data
2. **Dynamic**: Filters applied at runtime based on env vars
3. **Automatic**: shard_arr generated from total_shards
4. **Extensible**: Add new component = add dict entry

---

## Visualizing the Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONFIGURATION → MATRIX → JOBS                         │
└─────────────────────────────────────────────────────────────────────────┘

Configuration Phase:
────────────────────
┌──────────────────────────────────────────────────────────┐
│ test_matrix = {                                          │
│   "rocblas":  {total_shards: 6, timeout: 288, ...},     │
│   "miopen":   {total_shards: 4, timeout: 120, ...},     │
│   "rocrand":  {total_shards: 1, timeout: 15, ...},      │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
                    │
                    │ fetch_test_configurations.py
                    │ + Environment filters
                    ▼
Matrix Generation:
──────────────────
┌──────────────────────────────────────────────────────────┐
│ components: [                                            │
│   {name: "rocblas",  shard_arr: [1,2,3,4,5,6], ...},    │
│   {name: "miopen",   shard_arr: [1,2,3,4], ...},        │
│   {name: "rocrand",  shard_arr: [1], ...},              │
│ ]                                                        │
└──────────────────────────────────────────────────────────┘
                    │
                    │ GitHub Actions workflow
                    │ strategy.matrix iteration
                    ▼
Job Expansion:
──────────────
┌──────────────────────────────────────────────────────────┐
│ Component Loop:                                          │
│   for component in [rocblas, miopen, rocrand]:          │
│                                                          │
│     Shard Loop (for that component):                    │
│       for shard in component.shard_arr:                 │
│         create_job(component, shard)                    │
│                                                          │
│ Results:                                                 │
│   rocblas: [1/6, 2/6, 3/6, 4/6, 5/6, 6/6]              │
│   miopen:  [1/4, 2/4, 3/4, 4/4]                        │
│   rocrand: [1/1]                                        │
│                                                          │
│ Total: 11 jobs                                           │
└──────────────────────────────────────────────────────────┘
                    │
                    │ GitHub Actions scheduler
                    │ Matches jobs to runners
                    ▼
Parallel Execution:
───────────────────
┌──────────────────────────────────────────────────────────┐
│ All 11 jobs run in parallel (subject to runner          │
│ availability):                                           │
│                                                          │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│ │rocblas   │ │rocblas   │ │miopen    │ │rocrand   │   │
│ │  1/6     │ │  2/6     │ │  1/4     │ │  1/1     │   │
│ └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
│                                                          │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐                 │
│ │rocblas   │ │miopen    │ │miopen    │ ...             │
│ │  3/6     │ │  2/4     │ │  3/4     │                 │
│ └──────────┘ └──────────┘ └──────────┘                 │
│                                                          │
│ Each job runs independently with its own SHARD_INDEX    │
└──────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

### 1. Configuration is Data, Not Code

```python
# This is pure configuration (data):
test_matrix = {
    "rocblas": {"total_shards_dict": {"linux": 6}},
}

# Not code like:
def get_rocblas_shards():
    if platform == "linux":
        return 6
```

**Benefits:**
- Easy to understand
- Easy to modify
- No logic bugs
- Can be validated programmatically

### 2. Matrix Expansion is Automatic

**You write:**
```python
"total_shards_dict": {"linux": 6}
```

**GitHub Actions creates:**
- 6 separate jobs
- All with same config except SHARD_INDEX
- All running in parallel
- All independent

**You don't write:**
- Job definitions for each shard
- Loops or scripts to create jobs
- Coordination logic between shards

### 3. Configuration Drives Multiple Dimensions

One configuration controls:
- **Component selection**: Which tests to run
- **Platform selection**: Linux vs Windows
- **Sharding**: How many parallel jobs
- **Timeouts**: Per-component limits
- **Container images**: Per-component environments
- **Artifacts**: What to download
- **GPU exclusions**: Architecture filtering

All from one `test_matrix` dictionary!

### 4. Developers Don't Manage Job Dispatch

**Developers do:**
- Add components to `test_matrix`
- Configure `total_shards` for desired parallelism
- Specify timeouts and platforms

**GitHub Actions does:**
- Create N jobs from `shard_arr`
- Schedule jobs to available runners
- Handle retries and failures
- Collect results

**No manual dispatching, no runner selection, no job coordination!**

---

## Summary

TheRock's configuration-driven test matrix provides:

✓ **Declarative**: State what you want, not how to achieve it
✓ **Automatic**: Matrix expansion and job creation handled by GitHub
✓ **Parallel**: Multiple shards per component, multiple components in parallel
✓ **Scalable**: Adjust `total_shards` to control parallelism
✓ **Flexible**: Filter by project, platform, GPU, test type
✓ **Maintainable**: Single source of truth in `test_matrix`
✓ **Extensible**: Add component = add dictionary entry

All powered by `fetch_test_configurations.py` reading `test_matrix` and generating JSON for GitHub Actions!
