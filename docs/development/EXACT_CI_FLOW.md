# Exact CI Flow: How configure_ci.py, BUILD_TOPOLOGY.toml, and amdgpu_family_matrix.py Interact

## The Critical Question

**Q: Are GPU-independent builds (like compiler) built once and reused, or built multiple times in each GPU-specific job?**

**A: Built ONCE and REUSED via S3 artifact sharing!**

---

## The Three Key Files

### 1. `amdgpu_family_matrix.py` - GPU List (Python Dictionary)

**Location:** `build_tools/github_actions/amdgpu_family_matrix.py`

**What it contains:**

```python
# This is pure Python code, NOT a config file!

amdgpu_family_info_matrix_presubmit = {
    "gfx94x": {
        "linux": {
            "test-runs-on": "linux-mi325-1gpu-ossci-rocm",  # Which runner machine
            "family": "gfx94X-dcgpu",                       # → Passed to CMake as -DTHEROCK_AMDGPU_FAMILIES
            "fetch-gfx-targets": ["gfx942"],                # For split artifacts
            "build_variants": ["release", "asan", "tsan"],  # Which build types
        }
    },
    "gfx110x": {
        "linux": {
            "test-runs-on": "linux-gfx110X-gpu-rocm",
            "family": "gfx110X-all",
            "fetch-gfx-targets": ["gfx1100"],
            "build_variants": ["release"],
        }
    },
    "gfx950": {
        "linux": {
            "test-runs-on": "linux-gfx950-gpu-rocm",
            "family": "gfx950-dcgpu",
            "fetch-gfx-targets": ["gfx950"],
            "build_variants": ["release"],
        }
    },
}

# Nightly builds include MORE GPUs
amdgpu_family_info_matrix_nightly = {
    # ... includes all from presubmit, postsubmit, plus more
}
```

**Purpose:** Defines which GPU families exist and where to test them.

### 2. `BUILD_TOPOLOGY.toml` - Build Structure (TOML Config)

**Location:** `/BUILD_TOPOLOGY.toml`

**What it contains:**

```toml
# Build stages - defines CI job structure
[build_stages.foundation]
artifact_groups = ["third-party-sysdeps", "base"]
# type not specified = "generic" (default)

[build_stages.compiler-runtime]
artifact_groups = ["compiler", "core-runtime", "hip-runtime"]
# type not specified = "generic"

[build_stages.math-libs]
artifact_groups = ["math-libs", "ml-libs"]
type = "per-arch"  # ← This tells CI to multiply jobs!

# Artifact groups - defines dependencies
[artifact_groups.compiler]
type = "generic"  # ← This means: build once for all GPUs
artifact_group_deps = ["third-party-sysdeps"]

[artifact_groups.math-libs]
type = "per-arch"  # ← This means: build separately per GPU
artifact_group_deps = ["hip-runtime"]
```

**Purpose:** Defines build structure, dependencies, and whether stages are generic or per-arch.

### 3. `configure_ci.py` - The Orchestrator (Python Script)

**Location:** `build_tools/github_actions/configure_ci.py`

**What it does:** Reads amdgpu_family_matrix.py and outputs a GitHub Actions matrix.

**IMPORTANT:** It does NOT read BUILD_TOPOLOGY.toml directly! (That's read later by configure_stage.py)

**Process:**

```python
# Simplified logic from configure_ci.py

from amdgpu_family_matrix import get_all_families_for_trigger_types

# Step 1: Determine trigger type (presubmit, postsubmit, nightly)
if is_pull_request:
    trigger_types = ["presubmit"]
elif is_schedule:  # Nightly
    trigger_types = ["presubmit", "postsubmit", "nightly"]

# Step 2: Get GPU families from amdgpu_family_matrix.py
families = get_all_families_for_trigger_types(trigger_types, platform="linux")
# Returns: ["gfx94X-dcgpu", "gfx1100", "gfx950-dcgpu"]

# Step 3: Create matrix JSON
matrix_output = []
for family in families:
    matrix_output.append({
        "amdgpu_family": family,
        "test-runs-on": get_runner_label(family),
    })

# Step 4: Output JSON for GitHub Actions
print(f"linux_variants={json.dumps(matrix_output)}")
# Output: [
#   {"amdgpu_family": "gfx94X-dcgpu", "test-runs-on": "linux-mi325-1gpu"},
#   {"amdgpu_family": "gfx1100", "test-runs-on": "linux-gfx110X-gpu"},
#   {"amdgpu_family": "gfx950-dcgpu", "test-runs-on": "linux-gfx950-gpu"},
# ]
```

---

## The Complete CI Pipeline Flow

### Stage 1: GitHub Actions Workflow Starts

```yaml
# .github/workflows/multi_arch_build_portable_linux.yml

on:
  pull_request:  # or push, or schedule for nightly

jobs:
  # This calls setup.yml which runs configure_ci.py
  setup:
    uses: ./.github/workflows/setup.yml
    # configure_ci.py outputs: linux_variants JSON

  build:
    needs: setup
    uses: ./.github/workflows/multi_arch_build_portable_linux.yml
    with:
      matrix_per_family_json: ${{ needs.setup.outputs.linux_variants }}
      dist_amdgpu_families: "gfx94X-dcgpu;gfx1100;gfx950-dcgpu"
```

### Stage 2: Multi-Arch Build Workflow Structure

```yaml
# .github/workflows/multi_arch_build_portable_linux.yml

jobs:
  # ==========================================================================
  # STAGE: foundation (GENERIC - runs ONCE for all GPUs)
  # ==========================================================================
  foundation:
    uses: ./.github/workflows/multi_arch_build_portable_linux_artifacts.yml
    with:
      stage_name: foundation
      amdgpu_family: ""  # Empty = generic build
      dist_amdgpu_families: "gfx94X-dcgpu;gfx1100;gfx950-dcgpu"
    # This job:
    #   1. Runs cmake with ALL GPU families
    #   2. Builds sysdeps, base (generic artifacts)
    #   3. Uploads to S3: therock-base-linux.tar.xz, therock-sysdeps-linux.tar.xz

  # ==========================================================================
  # STAGE: compiler-runtime (GENERIC - runs ONCE for all GPUs)
  # ==========================================================================
  compiler-runtime:
    needs: foundation  # Waits for foundation
    uses: ./.github/workflows/multi_arch_build_portable_linux_artifacts.yml
    with:
      stage_name: compiler-runtime
      amdgpu_family: ""  # Empty = generic build
      dist_amdgpu_families: "gfx94X-dcgpu;gfx1100;gfx950-dcgpu"
    # This job:
    #   1. Downloads foundation artifacts from S3
    #   2. Runs cmake with ALL GPU families
    #   3. Builds compiler, HIP runtime (generic artifacts)
    #   4. Uploads to S3: therock-compiler-linux.tar.xz, therock-hip-runtime-linux.tar.xz

  # ==========================================================================
  # STAGE: math-libs (PER-ARCH - runs 3 TIMES in PARALLEL)
  # ==========================================================================
  math-libs:
    needs: compiler-runtime  # Waits for compiler-runtime
    strategy:
      matrix:
        family_info: ${{ fromJSON(inputs.matrix_per_family_json) }}
        # This creates 3 parallel jobs:
        # Job 1: family_info = {amdgpu_family: "gfx94X-dcgpu"}
        # Job 2: family_info = {amdgpu_family: "gfx1100"}
        # Job 3: family_info = {amdgpu_family: "gfx950-dcgpu"}
    uses: ./.github/workflows/multi_arch_build_portable_linux_artifacts.yml
    with:
      stage_name: math-libs
      amdgpu_family: ${{ matrix.family_info.amdgpu_family }}  # Different per job!
      dist_amdgpu_families: "gfx94X-dcgpu;gfx1100;gfx950-dcgpu"
    # Job 1 (gfx94X):
    #   1. Downloads compiler-runtime artifacts from S3
    #   2. Runs cmake -DTHEROCK_AMDGPU_FAMILIES=gfx94X-dcgpu
    #   3. Builds rocBLAS, rocFFT, etc. for MI300 GPUs
    #   4. Uploads to S3: therock-blas-linux-gfx94X-dcgpu.tar.xz
    #
    # Job 2 (gfx1100) - runs AT THE SAME TIME on different machine:
    #   1. Downloads compiler-runtime artifacts from S3 (same artifacts!)
    #   2. Runs cmake -DTHEROCK_AMDGPU_FAMILIES=gfx1100
    #   3. Builds rocBLAS, rocFFT, etc. for RX 7000 GPUs
    #   4. Uploads to S3: therock-blas-linux-gfx1100.tar.xz
    #
    # Job 3 (gfx950) - runs AT THE SAME TIME on different machine:
    #   1. Downloads compiler-runtime artifacts from S3 (same artifacts!)
    #   2. Runs cmake -DTHEROCK_AMDGPU_FAMILIES=gfx950-dcgpu
    #   3. Builds rocBLAS, rocFFT, etc. for MI340 GPUs
    #   4. Uploads to S3: therock-blas-linux-gfx950-dcgpu.tar.xz
```

### Stage 3: Inside Each Build Job

```bash
# .github/workflows/multi_arch_build_portable_linux_artifacts.yml

# STEP 1: Fetch inbound artifacts from previous stages
python build_tools/artifact_manager.py fetch \
  --run-id=${{ github.run_id }} \
  --stage="math-libs" \
  --amdgpu-families="gfx94X-dcgpu" \
  --output-dir="build" \
  --bootstrap

# This downloads from S3:
# build/artifacts/therock-compiler-linux.tar.xz          (built once in compiler-runtime)
# build/artifacts/therock-hip-runtime-linux.tar.xz       (built once in compiler-runtime)
# build/artifacts/therock-base-linux.tar.xz              (built once in foundation)
# build/artifacts/therock-sysdeps-linux.tar.xz           (built once in foundation)

# STEP 2: Fetch git sources
./build_tools/fetch_sources.py --stage math-libs

# This reads BUILD_TOPOLOGY.toml:
#   [build_stages.math-libs]
#   artifact_groups = ["math-libs"]
#
#   [artifact_groups.math-libs]
#   source_sets = ["rocm-libraries"]
#
#   [source_sets.rocm-libraries]
#   submodules = ["rocm-libraries"]
#
# Result: Only clones rocm-libraries submodule

# STEP 3: Get stage configuration
python build_tools/configure_stage.py \
  --stage="math-libs" \
  --amdgpu-families="gfx94X-dcgpu" \
  --gha-output

# This reads BUILD_TOPOLOGY.toml and outputs CMake args:
# cmake_args=-DTHEROCK_ENABLE_MATH_LIBS=ON -DTHEROCK_ENABLE_ML_LIBS=ON -DTHEROCK_AMDGPU_FAMILIES=gfx94X-dcgpu

# STEP 4: Configure with CMake
cmake -B build -GNinja \
  -DTHEROCK_ENABLE_MATH_LIBS=ON \
  -DTHEROCK_ENABLE_ML_LIBS=ON \
  -DTHEROCK_AMDGPU_FAMILIES=gfx94X-dcgpu

# CMake runs topology_to_cmake.py which generates:
#   THEROCK_ENABLE_BLAS=ON
#   THEROCK_ENABLE_FFT=ON
#   etc.

# STEP 5: Build
cmake --build build --target stage-math-libs therock-artifacts

# Builds:
#   rocBLAS for gfx94X
#   rocFFT for gfx94X
#   rocRAND for gfx94X
#   MIOpen for gfx94X
#
# Creates:
#   build/artifacts/therock-blas-linux-gfx94X-dcgpu.tar.xz
#   build/artifacts/therock-fft-linux-gfx94X-dcgpu.tar.xz
#   build/artifacts/therock-rand-linux-gfx94X-dcgpu.tar.xz
#   build/artifacts/therock-miopen-linux-gfx94X-dcgpu.tar.xz

# STEP 6: Upload to S3
python build_tools/artifact_manager.py push \
  --run-id=${{ github.run_id }} \
  --stage="math-libs" \
  --amdgpu-families="gfx94X-dcgpu" \
  --artifact-dir="build/artifacts"
```

---

## How BUILD_TOPOLOGY.toml is Actually Used

### By configure_stage.py (NOT configure_ci.py!)

```python
# build_tools/configure_stage.py

from _therock_utils.build_topology import BuildTopology

# Read BUILD_TOPOLOGY.toml
topology = BuildTopology("BUILD_TOPOLOGY.toml")

# Get stage info
stage = topology.build_stages["math-libs"]
artifact_groups = stage.artifact_groups  # ["math-libs", "ml-libs"]

# Determine which features to enable
features_to_enable = []
for group_name in artifact_groups:
    group = topology.artifact_groups[group_name]
    for artifact in topology.get_artifacts_in_group(group_name):
        features_to_enable.append(f"THEROCK_ENABLE_{artifact.feature_name}")

# Output CMake args
cmake_args = " ".join([f"-D{feature}=ON" for feature in features_to_enable])
cmake_args += f" -DTHEROCK_AMDGPU_FAMILIES={amdgpu_families}"

print(f"cmake_args={cmake_args}")
```

### By fetch_sources.py

```python
# build_tools/fetch_sources.py

from _therock_utils.build_topology import BuildTopology

topology = BuildTopology("BUILD_TOPOLOGY.toml")

# Get stage
stage = topology.build_stages["math-libs"]

# Get all source sets needed
source_sets = set()
for group_name in stage.artifact_groups:
    group = topology.artifact_groups[group_name]
    source_sets.update(group.source_sets)

# Download only those submodules
for source_set_name in source_sets:
    source_set = topology.source_sets[source_set_name]
    for submodule in source_set.submodules:
        run(f"git submodule update --init {submodule}")
```

---

## The Answer: Generic Builds Are Built ONCE

### Proof from S3 Artifact Names

**Generic artifacts (NO GPU suffix):**
```
s3://therock-ci-artifacts/12345-linux/
├── therock-compiler-linux.tar.xz          ← Built once by compiler-runtime job
├── therock-hip-runtime-linux.tar.xz       ← Built once by compiler-runtime job
├── therock-base-linux.tar.xz              ← Built once by foundation job
├── therock-sysdeps-linux.tar.xz           ← Built once by foundation job
```

**Per-arch artifacts (WITH GPU suffix):**
```
├── therock-blas-linux-gfx94X-dcgpu.tar.xz ← Built by math-libs job 1
├── therock-blas-linux-gfx1100.tar.xz      ← Built by math-libs job 2
├── therock-blas-linux-gfx950-dcgpu.tar.xz ← Built by math-libs job 3
├── therock-fft-linux-gfx94X-dcgpu.tar.xz  ← Built by math-libs job 1
├── therock-fft-linux-gfx1100.tar.xz       ← Built by math-libs job 2
├── therock-fft-linux-gfx950-dcgpu.tar.xz  ← Built by math-libs job 3
```

### The Flow

```
Time 0:00 - foundation job starts (1 machine)
  Builds: base, sysdeps
  Uploads: therock-base-linux.tar.xz, therock-sysdeps-linux.tar.xz
  Duration: 30 minutes

Time 0:30 - compiler-runtime job starts (1 machine)
  Downloads: therock-base-linux.tar.xz, therock-sysdeps-linux.tar.xz
  Builds: compiler, HIP runtime (for ALL GPU families)
  Uploads: therock-compiler-linux.tar.xz, therock-hip-runtime-linux.tar.xz
  Duration: 2 hours

Time 2:30 - math-libs jobs start (3 machines in parallel!)
  Job 1 (gfx94X):
    Downloads: therock-compiler-linux.tar.xz ← SAME FILE as Job 2 and 3!
    Builds: rocBLAS for gfx94X only
    Uploads: therock-blas-linux-gfx94X-dcgpu.tar.xz
    Duration: 3 hours

  Job 2 (gfx1100) - runs simultaneously:
    Downloads: therock-compiler-linux.tar.xz ← SAME FILE as Job 1 and 3!
    Builds: rocBLAS for gfx1100 only
    Uploads: therock-blas-linux-gfx1100.tar.xz
    Duration: 3 hours

  Job 3 (gfx950) - runs simultaneously:
    Downloads: therock-compiler-linux.tar.xz ← SAME FILE as Job 1 and 2!
    Builds: rocBLAS for gfx950 only
    Uploads: therock-blas-linux-gfx950-dcgpu.tar.xz
    Duration: 3 hours

Time 5:30 - All done!

Total time: 5.5 hours
Compiler built: ONCE (reused 3 times)
rocBLAS built: 3 TIMES (once per GPU family)
```

---

## Summary: The Three Actors

| File/Script | Role | What It Reads | What It Produces | When It Runs |
|---|---|---|---|---|
| **amdgpu_family_matrix.py** | GPU list | N/A (it's the data) | Python dict of GPUs | Imported by configure_ci.py |
| **configure_ci.py** | CI orchestrator | amdgpu_family_matrix.py | GitHub Actions matrix JSON | At CI workflow start |
| **BUILD_TOPOLOGY.toml** | Build structure | N/A (it's the data) | TOML config | Read by multiple scripts |
| **configure_stage.py** | CMake arg builder | BUILD_TOPOLOGY.toml | CMake arguments | In each build job |
| **fetch_sources.py** | Source downloader | BUILD_TOPOLOGY.toml | git submodules | In each build job |
| **topology_to_cmake.py** | CMake codegen | BUILD_TOPOLOGY.toml | CMake code | At `cmake -B build` time |

**Key insight:**
- `configure_ci.py` does NOT read BUILD_TOPOLOGY.toml
- It only reads `amdgpu_family_matrix.py` to create the GitHub Actions matrix
- BUILD_TOPOLOGY.toml is read later by scripts running INSIDE each build job
- Generic builds run ONCE and upload to S3
- Per-arch builds download generic artifacts from S3 and build GPU-specific code
