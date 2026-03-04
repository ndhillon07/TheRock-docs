# TheRock Packaging Deep Dive: From Source Code to End Users

## What This Document Covers

This guide explains how TheRock transforms source code into packages that end users can install. We'll follow the same path that CI takes every day:

1. **Source Code** → Build with CMake
2. **Build Output** → Portable artifacts (`.tar.xz` files)
3. **Portable Artifacts** → Three different package types:
   - Python wheels (for `pip install`)
   - Native Linux packages (for `apt install` / `yum install`)
   - Windows packages
4. **Packages** → Distribution via S3 and CloudFront
5. **End Users** → Install ROCm software

By the end, you'll understand how a single CMake build produces multiple package formats, where they go, and how users install them.

---

## Part 1: The Big Picture

### What is "Packaging"?

When we build TheRock from source, we get a directory tree with binaries, libraries, and headers:

```
build/dist/rocm/
├── bin/
│   ├── hipcc
│   ├── rocm-smi
│   └── rocblas-bench
├── lib/
│   ├── libamdhip64.so.6.2.0
│   ├── librocblas.so.4.0.0
│   └── cmake/
└── include/
    ├── hip/
    └── rocblas/
```

This directory works great if you built it yourself. But how do we give this to users who just want to run `pip install rocm` or `apt install rocm`?

That's what **packaging** does: it takes the build output and transforms it into formats that package managers understand.

### The Three Package Flavors

TheRock produces three main package types, all from the same build:

| Package Type | Package Manager | Platform | Example Install Command |
|---|---|---|---|
| **Python Wheels** | pip | Linux, Windows | `pip install rocm[libraries]` |
| **Native Linux** | apt / yum | Linux only | `apt install rocm` |
| **Windows Installers** | (future) | Windows only | (not yet implemented) |

**Key insight:** All three are built from the same source and produce functionally equivalent ROCm installations. The difference is just the delivery mechanism.

### The CI Packaging Pipeline

Here's how CI builds and packages TheRock every night:

```
┌─────────────────────────────────────────────────────────────┐
│ STAGE 0: SOURCE CODE                                        │
│ - Git repositories (submodules)                             │
│ - CMAKE configuration files                                 │
│ - BUILD_TOPOLOGY.toml (defines what to build)              │
└────────────────────────┬────────────────────────────────────┘
                         │
            ┌────────────┴─────────────┐
            │                          │
            ↓                          ↓
┌────────────────────────┐   ┌────────────────────────┐
│ STAGE 1A: BUILD LINUX  │   │ STAGE 1B: BUILD WINDOWS│
│ Duration: 4-8 hours    │   │ Duration: 4-8 hours    │
│ Output: Component      │   │ Output: Component      │
│   artifacts (.tar.xz)  │   │   artifacts (.tar.xz)  │
└───────────┬────────────┘   └───────────┬────────────┘
            │                            │
            │ Upload to S3               │ Upload to S3
            ↓                            ↓
┌─────────────────────────────────────────────────────────────┐
│ S3 STORAGE: therock-ci-artifacts/{run_id}/                  │
│ ├── therock-base-linux-gfx94X.tar.xz                       │
│ ├── therock-compiler-linux-gfx94X.tar.xz                   │
│ ├── therock-core-linux-gfx94X.tar.xz                       │
│ ├── therock-math-libs-linux-gfx94X.tar.xz                  │
│ └── (same for Windows)                                      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ Download artifacts
                         │
        ┌────────────────┼─────────────────┐
        │                │                 │
        ↓                ↓                 ↓
┌────────────┐   ┌────────────────┐   ┌──────────────┐
│ STAGE 2A:  │   │ STAGE 2B:      │   │ STAGE 2C:    │
│ PYTHON     │   │ PYTHON         │   │ NATIVE LINUX │
│ PACKAGES   │   │ PACKAGES       │   │ PACKAGES     │
│ (Linux)    │   │ (Windows)      │   │ (RPM + DEB)  │
│            │   │                │   │              │
│ Duration:  │   │ Duration:      │   │ Duration:    │
│ 10-20 min  │   │ 10-20 min      │   │ 20-40 min    │
└─────┬──────┘   └────────┬───────┘   └──────┬───────┘
      │                   │                  │
      │ Upload            │ Upload           │ Upload
      ↓                   ↓                  ↓
┌─────────────────────────────────────────────────────────────┐
│ S3 DISTRIBUTION STORAGE                                     │
│ ├── Python wheels (therock-nightly-python/)                │
│ ├── Native packages (therock-nightly-packages/)            │
│ └── Windows packages (future)                               │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ CloudFront CDN
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ END USERS                                                   │
│ pip install rocm[libraries]                                 │
│ apt install rocm                                            │
│ yum install rocm                                            │
└─────────────────────────────────────────────────────────────┘
```

**Key points:**

1. **Stage 1** (building) is slow (4-8 hours) and runs once per platform
2. **Stage 2** (packaging) is fast (10-40 minutes) and can run in parallel
3. All Stage 2 workflows download the same Stage 1 artifacts
4. This means we build once, package three ways

### Build Variants vs Package Types

Don't confuse these two concepts:

- **Build variant** = *How* we compile the code (release, debug, asan, tsan)
- **Package type** = *How* we deliver the compiled code (Python, RPM, DEB)

Any build variant can be packaged into any package type. For example:
- The nightly release build → Python wheels + RPM + DEB
- An ASAN debug build → Python wheels only (for testing)

---

## Part 2: Stage 0 - Source Code Organization

Before we build anything, let's understand how TheRock knows what to build and how it's organized.

### The Master Blueprint: BUILD_TOPOLOGY.toml

ROCm has dozens of components (compilers, libraries, tools). BUILD_TOPOLOGY.toml is the master configuration file that answers these questions:

- What are all the components?
- Which git repositories do we need?
- What depends on what?
- How should we group them for building and packaging?

**Location:** `/BUILD_TOPOLOGY.toml` (root of TheRock repository)

**Format:** TOML (Tom's Obvious Minimal Language) - a simple config file format

### Who Reads BUILD_TOPOLOGY.toml? (Critical Distinction!)

**CRITICAL: configure_ci.py does NOT read BUILD_TOPOLOGY.toml!**

Let's be very clear about what reads what:

| System | What Reads It | What It Reads | What It Does | Output |
|---|---|---|---|---|
| **GitHub Actions** | configure_ci.py | amdgpu_family_matrix.py ONLY | Creates parallel CI job matrix | JSON: which GPU families to build |
| **Build Jobs** | fetch_sources.py, configure_stage.py, etc. | BUILD_TOPOLOGY.toml | Determines what to build in each job | Submodule list, CMake flags |
| **CMake** | topology_to_cmake.py | BUILD_TOPOLOGY.toml | Generates build system | CMake targets, feature flags |

**Key insights:**
- **configure_ci.py** creates the matrix of parallel jobs from **amdgpu_family_matrix.py** (NOT BUILD_TOPOLOGY.toml!)
- **BUILD_TOPOLOGY.toml** is read LATER by scripts that run inside each build job
- **CMake doesn't create parallel jobs** - GitHub Actions does that
- Verify yourself: `grep BUILD_TOPOLOGY build_tools/github_actions/configure_ci.py` → No matches!

### How CMake Processes BUILD_TOPOLOGY.toml

**This happens on YOUR machine when you configure:**

```
┌─────────────────────────────────────────────────────────────┐
│ STEP 1: CMake Configure Time                                │
│ (happens when you run: cmake -B build)                      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ CMake executes a Python script:                             │
│                                                              │
│   execute_process(                                          │
│     COMMAND python3 build_tools/topology_to_cmake.py        │
│       --topology BUILD_TOPOLOGY.toml                        │
│       --output build/cmake/therock_topology.cmake           │
│   )                                                          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 2: Python Script Reads TOML and Generates CMake        │
│                                                              │
│ topology_to_cmake.py:                                       │
│   1. Parses BUILD_TOPOLOGY.toml                             │
│   2. Validates dependencies                                 │
│   3. Computes build order                                   │
│   4. Generates CMake code:                                  │
│      - therock_add_feature() calls (CMake function from     │
│        cmake/therock_features.cmake - creates ON/OFF flags) │
│      - CMake targets (artifact-blas, etc.)                  │
│      - Dependency variables                                 │
│      - THEROCK_ARTIFACT_TYPE_{name} variables               │
│                                                              │
│ ⚠️  CRITICAL: CMake does NOT natively understand TOML!      │
│ ⚠️  The Python script converts TOML → CMake variables       │
│                                                              │
│ Example generated CMake code:                                │
│   set(THEROCK_ARTIFACT_TYPE_blas "target-specific")         │
│   set(THEROCK_ARTIFACT_TYPE_compiler "target-neutral")      │
│   # (From BUILD_TOPOLOGY.toml artifacts[].type field)       │
│                                                              │
│ File: build_tools/topology_to_cmake.py                      │
│ Line 257: Converts artifact.type → CMake variable           │
│   f.write(f'set(THEROCK_ARTIFACT_TYPE_{name} "{type}")\n')  │
│                                                              │
│ Output: build/cmake/therock_topology.cmake                  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 3: CMake Includes Generated File                       │
│                                                              │
│   include(build/cmake/therock_topology.cmake)               │
│                                                              │
│ This creates:                                                │
│   - THEROCK_ENABLE_MATH_LIBS cache variable (ON/OFF)        │
│   - THEROCK_ENABLE_BLAS cache variable (ON/OFF)             │
│   - CMake targets: artifact-blas, stage-math-libs           │
│   - Build order list: THEROCK_BUILD_ORDER                   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 4: CMake Build System Uses Generated Info              │
│                                                              │
│ When you run: ninja -C build                                │
│   - Builds only enabled features                            │
│   - Respects dependency order                               │
│   - Creates artifact .tar.xz files                          │
└─────────────────────────────────────────────────────────────┘
```

**Key insights:**
- **BUILD_TOPOLOGY.toml is a data file, not a build script** - Python reads it once at configure time and translates it to CMake code
- **CMake doesn't natively understand TOML** - topology_to_cmake.py converts it to CMake `set()` commands
- **Build order is computed by Python** - topology_to_cmake.py analyzes dependencies and generates `THEROCK_BUILD_ORDER` list
- **`type = "per-arch"` becomes a CMake variable** - `THEROCK_ARTIFACT_TYPE_blas = "target-specific"`

**IMPORTANT: Understanding the "type" Field Terminology**

The `type` field uses different values depending on where it appears in BUILD_TOPOLOGY.toml:

| Location | Possible Values | Meaning |
|----------|----------------|---------|
| `[build_stages.*]` | `"generic"` or `"per-arch"` | How many times to build (once vs per-GPU) |
| `[artifact_groups.*]` | `"generic"` or `"per-arch"` | Same as build stages |
| `[artifacts.*]` | `"target-neutral"` or `"target-specific"` | GPU dependency (works-for-all vs GPU-specific) |

**Why different terminology?**
- Build stages/groups use **"generic"/"per-arch"** to describe BUILD BEHAVIOR (build once vs build N times)
- Artifacts use **"target-neutral"/"target-specific"** to describe ARTIFACT PROPERTIES (GPU dependency)
- Python script `topology_to_cmake.py` translates between them:
  - `"generic"` → `"target-neutral"` (build once, works for all GPUs)
  - `"per-arch"` → `"target-specific"` (build per GPU family)

**Example showing the translation:**

```toml
# BUILD_TOPOLOGY.toml
[build_stages.math-libs]
type = "per-arch"  # ← Build separately for each GPU family

[artifact_groups.math-libs]
type = "per-arch"  # ← Same: artifacts in this group are GPU-specific

[artifacts.blas]
type = "target-specific"  # ← Different terminology, same meaning
```

```cmake
# Generated by topology_to_cmake.py → cmake/therock_topology_generated.cmake
set(THEROCK_ARTIFACT_TYPE_blas "target-specific")  # ← Artifact type preserved as-is
```

**Mental model:** "generic"/"per-arch" = CI/build terminology, "target-neutral"/"target-specific" = packaging terminology

### How GitHub Actions Processes BUILD_TOPOLOGY.toml

**CRITICAL: configure_ci.py does NOT read BUILD_TOPOLOGY.toml!**

Let me show you the complete chain with actual file verification:

**This happens in CI when code is pushed to GitHub:**

```
┌─────────────────────────────────────────────────────────────┐
│ STEP 1: GitHub Actions Workflow Starts                      │
│ (e.g., when you push code or create a PR)                   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 2: Setup Job Runs configure_ci.py                      │
│                                                              │
│   - name: Configuring CI options                            │
│     run: ./build_tools/github_actions/configure_ci.py       │
│                                                              │
│ File: build_tools/github_actions/configure_ci.py            │
│ Line 57-58: Shows what it imports:                          │
│   from amdgpu_family_matrix import (                        │
│       all_build_variants,                                   │
│       get_all_families_for_trigger_types,                   │
│   )                                                          │
│                                                              │
│ ⚠️  DOES NOT IMPORT BUILD_TOPOLOGY - verify yourself!       │
│ ⚠️  Run: grep -n "BUILD_TOPOLOGY\|build_topology"           │
│           build_tools/github_actions/configure_ci.py        │
│     Result: No matches found                                │
│                                                              │
│ This script:                                                 │
│   1. Reads amdgpu_family_matrix.py (GPU families ONLY)      │
│   2. Creates a JSON matrix of build jobs                    │
│   3. Does NOT read BUILD_TOPOLOGY.toml at all!              │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 3: Matrix Output (Example for nightly build)           │
│                                                              │
│ linux_variants: [                                           │
│   {family: "gfx94X-dcgpu", variant: "release"},             │
│   {family: "gfx1100", variant: "release"},                  │
│   {family: "gfx950-dcgpu", variant: "release"}              │
│ ]                                                            │
│                                                              │
│ ⚠️  Notice: This only has GPU families, no build stages!    │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 4: GitHub Actions Creates Multiple Jobs                │
│                                                              │
│   build:                                                     │
│     strategy:                                                │
│       matrix: ${{ fromJson(needs.setup.outputs.linux_variants) }}│
│                                                              │
│ This creates 3 independent CI jobs:                         │
│   - Job 1: family=gfx94X-dcgpu   (Machine A in Azure)       │
│   - Job 2: family=gfx1100         (Machine B in Azure)       │
│   - Job 3: family=gfx950-dcgpu   (Machine C in Azure)       │
│                                                              │
│ All run SIMULTANEOUSLY on different machines!               │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ STEP 5: INSIDE Each Build Job - THIS is where               │
│         BUILD_TOPOLOGY.toml gets read!                       │
│                                                              │
│ Each job checks out code and runs:                          │
│                                                              │
│   - name: Fetch sources                                      │
│     run: ./build_tools/fetch_sources.py --jobs 12           │
│                                                              │
│ File: build_tools/fetch_sources.py                          │
│ Line 22: TOPOLOGY_PATH = THEROCK_DIR / "BUILD_TOPOLOGY.toml"│
│ Line 95-98: Opens and reads BUILD_TOPOLOGY.toml:            │
│   topology = BuildTopology(str(TOPOLOGY_PATH))              │
│                                                              │
│ ✓ Verify: grep -n "BUILD_TOPOLOGY" build_tools/fetch_sources.py│
│   Result: Line 22, 25, 95, etc. - FOUND!                    │
│                                                              │
│   - name: Configure Projects                                 │
│     run: python3 build_tools/github_actions/build_configure.py│
│                                                              │
│ This eventually calls:                                       │
│   cmake -B build -DTHEROCK_AMDGPU_FAMILIES=gfx94X-dcgpu     │
│                                                              │
│ Which triggers topology_to_cmake.py:                         │
│                                                              │
│ File: build_tools/topology_to_cmake.py                      │
│ Line 317: topology = BuildTopology(str(topology_path))      │
│                                                              │
│ ✓ Verify: grep -n "BuildTopology" build_tools/topology_to_cmake.py│
│   Result: Line 48, 317 - FOUND!                             │
│                                                              │
│   - name: Build therock-archives                             │
│     run: cmake --build build --target therock-archives       │
└─────────────────────────────────────────────────────────────┘
```

**The actual file chain - with verification commands:**

```python
# File 1: build_tools/github_actions/configure_ci.py
# Line 57-58 (verify: grep -n "^from amdgpu" configure_ci.py)
from amdgpu_family_matrix import (
    all_build_variants,
    get_all_families_for_trigger_types,
)

# ⚠️  Does NOT import BUILD_TOPOLOGY!
# Verify: grep -n "BUILD_TOPOLOGY\|BuildTopology" configure_ci.py
#   Result: No matches found


# Get GPU families from amdgpu_family_matrix.py (NOT from BUILD_TOPOLOGY.toml!)
families = get_all_families_for_trigger_types(["presubmit", "postsubmit", "nightly"])

# For each family, create a matrix entry
linux_variants = []
for family_name, family_config in families.items():
    linux_variants.append({
        "family": family_config["linux"]["family"],  # e.g., "gfx94X-dcgpu"
        "variant": "release"
    })

# Output as JSON for GitHub Actions
print(f"linux_variants={json.dumps(linux_variants)}")
```

```python
# build_tools/github_actions/amdgpu_family_matrix.py
# This is a Python dictionary, NOT from BUILD_TOPOLOGY.toml!

amdgpu_family_info_matrix_presubmit = {
    "gfx94x": {
        "linux": {
            "test-runs-on": "linux-mi325-1gpu-ossci-rocm",
            "family": "gfx94X-dcgpu",  # ← This is what gets passed to CMake!
            "fetch-gfx-targets": ["gfx942"],
            "build_variants": ["release", "asan", "tsan"],
        }
    },
    "gfx110x": {
        "linux": {
            "family": "gfx110X-all",
            "fetch-gfx-targets": ["gfx1100"],
        }
    },
}
```

**Critical point:** The GPU families (gfx94X-dcgpu, gfx1100, etc.) come from `amdgpu_family_matrix.py`, NOT from BUILD_TOPOLOGY.toml!

**BUILD_TOPOLOGY.toml only says `type = "per-arch"`** - it doesn't list which GPUs. GitHub Actions reads that and says "okay, I need to create one job per GPU family from my matrix."

### CRITICAL: Does configure_ci.py Actually Read BUILD_TOPOLOGY.toml?

**NO! This is a common misconception.**

**configure_ci.py ONLY reads amdgpu_family_matrix.py.** BUILD_TOPOLOGY.toml is read by completely different scripts that run later inside each build job.

**Here's the exact flow:**

```
GitHub Actions Workflow Starts
  ↓
setup.yml runs configure_ci.py
  ├─ Reads: amdgpu_family_matrix.py ONLY
  ├─ Outputs: JSON matrix of GPU families
  └─ Does NOT open BUILD_TOPOLOGY.toml
  ↓
GitHub Actions creates parallel jobs from matrix
  ↓
Each build job runs these scripts (which DO read BUILD_TOPOLOGY.toml):
  ├─ configure_stage.py → determines CMake flags from BUILD_TOPOLOGY.toml
  ├─ fetch_sources.py → downloads submodules based on BUILD_TOPOLOGY.toml
  └─ topology_to_cmake.py → generates CMake code from BUILD_TOPOLOGY.toml
```

**Complete file verification table:**

| Script | Reads BUILD_TOPOLOGY.toml? | Reads amdgpu_family_matrix.py? | Verification Command |
|--------|---------------------------|-------------------------------|---------------------|
| **configure_ci.py** | ❌ NO | ✅ YES | `grep -n "BUILD_TOPOLOGY" build_tools/github_actions/configure_ci.py` → No matches |
| **topology_to_cmake.py** | ✅ YES | ❌ NO | `grep -n "BuildTopology" build_tools/topology_to_cmake.py` → Lines 48, 317 |
| **configure_stage.py** | ✅ YES | ❌ NO | `grep -n "BUILD_TOPOLOGY" build_tools/configure_stage.py` → Lines 64, 68, 70 |
| **fetch_sources.py** | ✅ YES | ❌ NO | `grep -n "BUILD_TOPOLOGY" build_tools/fetch_sources.py` → Lines 22, 25, 95, etc. |
| **artifact_manager.py** | ✅ YES | ❌ NO | `grep -n "BUILD_TOPOLOGY" build_tools/artifact_manager.py` → Lines 20, 22, 24 |

**The two data sources serve different purposes:**

| Data Source | Contains | Used By | Purpose |
|-------------|----------|---------|---------|
| **amdgpu_family_matrix.py** | GPU families (gfx94X, gfx1100), test runners, build variants | configure_ci.py | GitHub Actions matrix creation - decides WHICH machines to run |
| **BUILD_TOPOLOGY.toml** | Build stages, artifact groups, artifacts, dependencies | Scripts inside build jobs | Determines WHAT to build and HOW to organize it |

### Are Generic Builds Repeated in Each Matrix Job?

**IT DEPENDS ON WHICH WORKFLOW!** There are TWO different workflows:

#### Workflow Type 1: Monolithic Builds (ci_nightly.yml → ci_linux.yml)

**Answer: YES, generic builds are REPEATED for each GPU family**

**WHY?** Because configure_ci.py doesn't read BUILD_TOPOLOGY.toml! It only knows about GPU families from amdgpu_family_matrix.py. It doesn't know which components are generic vs per-arch, so it creates simple GPU-based jobs where each builds EVERYTHING.

This is the workflow used for nightly CI and quick testing. Each GPU family gets a completely independent build.

**Actual flow from ci_nightly.yml:**

```yaml
# ci_nightly.yml creates a matrix of GPU families
jobs:
  linux_build_and_test:
    strategy:
      matrix:
        variant: [
          {family: "gfx94X-dcgpu"},
          {family: "gfx1100"},
          {family: "gfx950-dcgpu"}
        ]
    # For EACH GPU family, calls ci_linux.yml:
    uses: ./.github/workflows/ci_linux.yml
    with:
      amdgpu_families: ${{ matrix.variant.family }}
```

**What ci_linux.yml does:**

```yaml
# ci_linux.yml calls build_portable_linux_artifacts.yml
jobs:
  build_portable_linux_artifacts:
    uses: ./.github/workflows/build_portable_linux_artifacts.yml
    with:
      amdgpu_families: ${{ inputs.amdgpu_families }}  # ONE GPU family
```

**What build_portable_linux_artifacts.yml does:**

```yaml
# This workflow builds EVERYTHING in one monolithic job
steps:
  - name: Fetch sources
    run: ./build_tools/fetch_sources.py --jobs 12
    # ↑ Downloads ALL source code (no --stage flag = everything)

  - name: Configure Projects
    run: python3 build_tools/github_actions/build_configure.py

  - name: Build therock-archives and therock-dist
    run: cmake --build build --target therock-archives therock-dist
    # ↑ Builds EVERYTHING: foundation, compiler, math-libs, ALL in one go
```

**Timeline showing REPEATED builds:**

**Example: Building for 3 GPU families (gfx94X, gfx1100, gfx950)**

```
T+0:00  THREE jobs start SIMULTANEOUSLY (3 separate machines, 1 per GPU family):

        Machine A (gfx94X build):
          ├─ Builds: compiler (generic)           ← Build #1
          ├─ Builds: base (generic)               ← Build #1
          ├─ Builds: rocBLAS for gfx94X only
          └─ Uploads: therock-archives-gfx94X.tar.xz

        Machine B (gfx1100 build):
          ├─ Builds: compiler (generic)           ← Build #2 (DUPLICATE!)
          ├─ Builds: base (generic)               ← Build #2 (DUPLICATE!)
          ├─ Builds: rocBLAS for gfx1100 only
          └─ Uploads: therock-archives-gfx1100.tar.xz

        Machine C (gfx950 build):
          ├─ Builds: compiler (generic)           ← Build #3 (DUPLICATE!)
          ├─ Builds: base (generic)               ← Build #3 (DUPLICATE!)
          ├─ Builds: rocBLAS for gfx950 only
          └─ Uploads: therock-archives-gfx950.tar.xz

Result: Compiler built 3 TIMES (once per GPU family job)
        rocBLAS built 3 TIMES (once per GPU family)
        Total waste: 4 hours × 3 = 12 hours of duplicated compiler builds!
```

**Why does ci_nightly use this wasteful approach?**

- **Simplicity:** Each job is completely independent, easier to debug
- **Isolation:** One GPU's build failure doesn't affect others
- **Testing:** Catches integration bugs that might only appear with specific GPU targets
- **Speed for small changes:** If you only need one GPU, you get everything in one job

#### Workflow Type 2: Sharded Builds (multi_arch_build_portable_linux.yml)

**Answer: NO, generic builds are built ONCE and shared via S3**

This is the optimized workflow used for release builds and official artifacts.

**Actual flow from multi_arch_build_portable_linux.yml:**

```yaml
jobs:
  # Stage 1: foundation (generic) - runs on 1 machine
  foundation:
    uses: ./.github/workflows/multi_arch_build_portable_linux_artifacts.yml
    with:
      stage_name: foundation
      amdgpu_family: ""  # Empty = generic build for ALL GPUs

  # Stage 2: compiler-runtime (generic) - runs on 1 machine
  compiler-runtime:
    needs: foundation
    uses: ./.github/workflows/multi_arch_build_portable_linux_artifacts.yml
    with:
      stage_name: compiler-runtime
      amdgpu_family: ""  # Empty = generic build for ALL GPUs

  # Stage 3: math-libs (per-arch) - runs on 3 machines IN PARALLEL
  math-libs:
    needs: compiler-runtime
    strategy:
      matrix:
        family_info: [
          {amdgpu_family: "gfx94X-dcgpu"},
          {amdgpu_family: "gfx1100"},
          {amdgpu_family: "gfx950-dcgpu"}
        ]
    uses: ./.github/workflows/multi_arch_build_portable_linux_artifacts.yml
    with:
      stage_name: math-libs
      amdgpu_family: ${{ matrix.family_info.amdgpu_family }}
```

**What multi_arch_build_portable_linux_artifacts.yml does:**

```yaml
steps:
  # CRITICAL: This fetches dependencies from S3!
  - name: Fetch inbound artifacts
    run: |
      python build_tools/artifact_manager.py fetch \
        --run-id=${{ github.run_id }} \
        --stage="${STAGE_NAME}" \
        --amdgpu-families="${{ inputs.amdgpu_family }}" \
        --output-dir="${BUILD_DIR}" \
        --bootstrap
    # ↑ Downloads previously built stages from S3
    # For math-libs stage, this downloads compiler-runtime artifacts!

  - name: Fetch sources
    run: ./build_tools/fetch_sources.py --stage ${STAGE_NAME}
    # ↑ Only downloads sources for THIS stage (not everything)

  - name: Build stage
    run: cmake --build build --target stage-${STAGE_NAME}
    # ↑ Only builds THIS stage (not everything)

  - name: Push stage artifacts
    run: |
      python build_tools/artifact_manager.py push \
        --run-id ${{ github.run_id }} \
        --stage="${STAGE_NAME}" \
        --build-dir="${BUILD_DIR}"
    # ↑ Uploads this stage's artifacts to S3 for next stages to use
```

**Timeline showing artifact reuse:**

**Example: Building for 3 GPU families (gfx94X, gfx1100, gfx950)**

```
T+0:00  foundation (Machine A)
        ├─ Builds: base, sysdeps (generic for ALL GPUs)
        └─ Uploads to S3: therock-base-linux.tar.xz

T+2:00  compiler-runtime (Machine B)
        ├─ Downloads from S3: therock-base-linux.tar.xz ← FROM PREVIOUS STAGE
        ├─ Builds: compiler, HIP runtime (generic for ALL GPUs)
        └─ Uploads to S3: therock-compiler-linux.tar.xz

T+4:00  THREE math-libs jobs start SIMULTANEOUSLY (1 per GPU family):

        Machine C (gfx94X):
          ├─ Downloads from S3: therock-compiler-linux.tar.xz ← SAME FILE
          ├─ Builds: rocBLAS for gfx94X ONLY
          └─ Uploads to S3: therock-blas-linux-gfx94X.tar.xz

        Machine D (gfx1100):
          ├─ Downloads from S3: therock-compiler-linux.tar.xz ← SAME FILE
          ├─ Builds: rocBLAS for gfx1100 ONLY
          └─ Uploads to S3: therock-blas-linux-gfx1100.tar.xz

        Machine E (gfx950):
          ├─ Downloads from S3: therock-compiler-linux.tar.xz ← SAME FILE
          ├─ Builds: rocBLAS for gfx950 ONLY
          └─ Uploads to S3: therock-blas-linux-gfx950.tar.xz

Result: Compiler built ONCE (on Machine B)
        Downloaded 3 TIMES (Machines C, D, E reuse it)
        rocBLAS built 3 TIMES (once per GPU)
```

**S3 bucket structure:**

```
s3://therock-ci-artifacts/{github.run_id}-linux/

Generic (built once, no GPU suffix):
  foundation/
    therock-base-linux.tar.xz          ← Built by foundation stage
  compiler-runtime/
    therock-compiler-linux.tar.xz      ← Built by compiler-runtime stage
    therock-hip-runtime-linux.tar.xz

Per-arch (built per GPU, with GPU suffix):
  math-libs/
    therock-blas-linux-gfx94X-dcgpu.tar.xz    ← Built by math-libs[gfx94X] job
    therock-blas-linux-gfx1100.tar.xz         ← Built by math-libs[gfx1100] job
    therock-blas-linux-gfx950-dcgpu.tar.xz    ← Built by math-libs[gfx950] job
```

**Efficiency comparison:**

```
Workflow Type 1 (Monolithic):
  Compiler builds: 2 hours × 3 GPUs = 6 hours total compute time
  rocBLAS builds:  1 hour  × 3 GPUs = 3 hours total compute time
  Total compute:                      9 hours
  Wall-clock time: 3 hours (parallel, but each job takes full 3 hours)

Workflow Type 2 (Sharded):
  Compiler builds: 2 hours × 1 = 2 hours total compute time
  rocBLAS builds:  1 hour  × 3 = 3 hours total compute time (parallel)
  Total compute:                 5 hours
  Wall-clock time: 4 hours (sequential: foundation→compiler→math-libs)

Savings: 4 hours of compute time
Trade-off: +1 hour wall-clock (sequential stages vs full parallel)
```

**Which workflow is used when?**

| Workflow | Trigger | When Used | Trade-off |
|----------|---------|-----------|-----------|
| Monolithic (ci_nightly) | Nightly, PR testing | Quick iteration, testing | Wastes compute, faster for single GPU |
| Sharded (multi_arch) | Release builds, official packages | Production artifacts | Saves compute, slower wall-clock |

### The 4-Level Hierarchy

BUILD_TOPOLOGY.toml organizes ROCm using four levels of abstraction:

```
LEVEL 1: Source Sets      (where is the code?)
   ↓
LEVEL 2: Build Stages     (how do we organize CI/CD jobs?)
   ↓
LEVEL 3: Artifact Groups  (how do we group related components?)
   ↓
LEVEL 4: Artifacts        (what are the actual .tar.xz files?)
```

**Visual Schematic with Relationships:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│ LEVEL 1: Source Sets (Git Submodules)                                  │
│ Cardinality: Many submodules → Many source sets                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  [source_sets.compilers]                [source_sets.rocm-libraries]   │
│   └─ submodules: ["llvm-project",       └─ submodules: ["rocm-libraries"]│
│                   "hipify", ...]                                        │
│                                                                         │
│   1 source set contains N submodules (1:N relationship)                │
└────────────────────────┬────────────────────────────────────────────────┘
                         │ referenced by
                         ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ LEVEL 2: Build Stages (CI Job Boundaries)                              │
│ Cardinality: 1 stage contains N artifact groups                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  [build_stages.foundation]         [build_stages.compiler-runtime]     │
│   ├─ artifact_groups: ["base",     ├─ artifact_groups: ["compiler",   │
│   │    "third-party-sysdeps"]      │    "core-runtime", "hip-runtime"]│
│   └─ type: "generic"               └─ type: "generic"                 │
│                                                                         │
│  [build_stages.math-libs]                                              │
│   ├─ artifact_groups: ["math-libs", "ml-libs"]                         │
│   └─ type: "per-arch" ← Built once per GPU family                     │
│                                                                         │
│   1 build stage contains N artifact groups (1:N relationship)          │
└────────────────────────┬────────────────────────────────────────────────┘
                         │ contains
                         ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ LEVEL 3: Artifact Groups (Logical Groupings)                           │
│ Cardinality: 1 group contains N artifacts                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  [artifact_groups.compiler]         [artifact_groups.math-libs]        │
│   ├─ type: "generic"                ├─ type: "target-specific"         │
│   ├─ source_sets: ["compilers"]     ├─ source_sets: ["rocm-libraries"]│
│   ├─ artifact_group_deps: []        ├─ artifact_group_deps:            │
│   └─ artifacts in this group:       │    ["compiler", "hip-runtime"]  │
│       • compiler                    └─ artifacts in this group:        │
│       • device-libs                     • blas                         │
│       • hipcc                           • fft                          │
│                                         • solver                       │
│                                                                         │
│   1 artifact group contains N artifacts (1:N relationship)             │
│   1 artifact group depends on M other groups (M:N relationship)        │
│   1 artifact group references K source sets (M:N relationship)         │
└────────────────────────┬────────────────────────────────────────────────┘
                         │ contains
                         ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ LEVEL 4: Artifacts (Individual .tar.xz files)                          │
│ Cardinality: 1 artifact can depend on N other artifacts                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  [artifacts.blas]                                                      │
│   ├─ type: "target-specific" ← Built per GPU family                   │
│   ├─ artifact_group: "math-libs"                                       │
│   ├─ artifact_deps: ["hip-runtime", "compiler"] ← Needs these first!  │
│   ├─ split_databases: ["rocblas"] ← Creates multiple .tar.xz files    │
│   └─ Output files:                                                     │
│       • therock-blas-linux-gfx94X-dcgpu.tar.xz                         │
│       • therock-blas-linux-gfx1100.tar.xz                              │
│       • therock-blas-linux-gfx950-dcgpu.tar.xz                         │
│                                                                         │
│  [artifacts.compiler]                                                  │
│   ├─ type: "target-neutral" ← Built once for all GPUs                 │
│   ├─ artifact_group: "compiler"                                        │
│   ├─ artifact_deps: ["third-party-sysdeps"]                            │
│   └─ Output file:                                                      │
│       • therock-compiler-linux.tar.xz (no GPU suffix!)                 │
│                                                                         │
│   1 artifact depends on N other artifacts (1:N relationship)           │
│   1 artifact belongs to exactly 1 artifact group (N:1 relationship)    │
└─────────────────────────────────────────────────────────────────────────┘

Key Relationships Summary:
  • 1 Build Stage → N Artifact Groups (contains)
  • 1 Artifact Group → N Artifacts (contains)
  • 1 Artifact Group → M Source Sets (references for code)
  • 1 Artifact Group → K Artifact Groups (depends on)
  • 1 Artifact → L Artifacts (depends on)
  • 1 Source Set → P Git Submodules (contains)
```

Let's understand each level with concrete examples.

### Level 1: Source Sets

#### What is a Git Submodule? (Starting from Basics)

Think of TheRock like a cookbook that references other cookbooks. The main cookbook doesn't contain all recipes - it just says "for desserts, see the Dessert Cookbook at this location."

In git terms:
- **TheRock repository** = main cookbook
- **Git submodules** = references to other cookbooks (other git repositories)
- **`.gitmodules` file** = table of contents listing all referenced cookbooks

**Real example from TheRock:**

The file `.gitmodules` in the root of TheRock contains:

```ini
[submodule "rocm-libraries"]
	path = rocm-libraries
	url = https://github.com/ROCm/rocm-libraries
	branch = develop

[submodule "llvm-project"]
	path = compiler/amd-llvm
	url = https://github.com/ROCm/llvm-project.git
	branch = amd-mainline

[submodule "HIPIFY"]
	path = compiler/hipify
	url = https://github.com/ROCm/HIPIFY.git
	branch = amd-mainline
```

**What this means:**

- **Submodule name:** `rocm-libraries` (this is how we refer to it)
- **Path:** Where it gets cloned inside TheRock → `rocm-libraries/` directory
- **URL:** Where to download it from → GitHub repository
- **Branch:** Which version to use → `develop` branch

**Analogy:**

```
Your main project (TheRock):
└── Like a main building with references to other buildings

Git submodules (.gitmodules):
└── Like sticky notes saying:
    - "Math library code is at 123 GitHub Street, use version 2.0"
    - "Compiler code is at 456 GitHub Avenue, use version 3.1"
```

#### What Are Source Sets?

**Problem:** TheRock has 20+ submodules. Downloading all of them takes time and disk space. If you only want to build the compiler, why download math libraries?

**Solution:** Source sets group related submodules.

**Real example from BUILD_TOPOLOGY.toml:**

```toml
[source_sets.compilers]
description = "Compiler toolchain submodules"
submodules = ["llvm-project", "HIPIFY", "spirv-llvm-translator"]

[source_sets.rocm-libraries]
description = "ROCm libraries monorepo (math libs)"
submodules = ["rocm-libraries"]

[source_sets.math-libs]
description = "Additional math library submodules"
submodules = ["libhipcxx"]
```

**The connection to .gitmodules:**

Notice that the names in `submodules = [...]` match the `[submodule "name"]` entries in `.gitmodules`:

```
BUILD_TOPOLOGY.toml says:     .gitmodules says:
submodules = ["llvm-project"]  →  [submodule "llvm-project"]
                                    path = compiler/amd-llvm
                                    url = https://github.com/ROCm/llvm-project.git
```

**How to use source sets with the fetch_sources.py script:**

```bash
# Scenario 1: Download everything (all 20+ submodules)
./build_tools/fetch_sources.py

# This runs:
git submodule update --init --recursive
# Result: Downloads all submodules listed in .gitmodules

# Scenario 2: Download only compiler sources
./build_tools/fetch_sources.py --stage compiler-runtime
                                       ^^^^^^^^^^^^^^^^
                                       This is a build stage name!
```

**Where does "compiler-runtime" come from?**

It's the name of a **build stage** defined later in BUILD_TOPOLOGY.toml.

**Quick preview** (we'll explain build stages in detail in "Level 2: Build Stages" below):
- A **build stage** is a group of components that are built together in one CI job
- It defines CI job boundaries and parallelization opportunities
- Example stages: foundation (runs first), compiler-runtime (runs second), math-libs (runs third)

Here's how it's defined in BUILD_TOPOLOGY.toml:

```toml
# BUILD_TOPOLOGY.toml (we'll explain build stages in detail later)

[build_stages.compiler-runtime]    ← The name "compiler-runtime" comes from here!
                ^^^^^^^^^^^^^^^^
description = "Compiler, runtimes, and core profiling"
artifact_groups = ["compiler", "core-runtime", "hip-runtime", "profiler-core"]
```

**The connection:**

```
Command line:
  ./build_tools/fetch_sources.py --stage compiler-runtime
                                           ↓
                        Looks up this name in BUILD_TOPOLOGY.toml
                                           ↓
BUILD_TOPOLOGY.toml:
  [build_stages.compiler-runtime]
  artifact_groups = ["compiler", ...]
                         ↓
                    Looks up which source_sets the "compiler" artifact_group needs
                         ↓
  [artifact_groups.compiler]
  source_sets = ["compilers"]  ← Needs the "compilers" source set
                         ↓
                    Downloads those submodules
                         ↓
  [source_sets.compilers]
  submodules = ["llvm-project", "HIPIFY", "spirv-llvm-translator"]
                         ↓
               Only downloads these 3 submodules from .gitmodules
```

**What actually happens:**

```bash
./build_tools/fetch_sources.py --stage compiler-runtime

# Script does:
#   1. Reads BUILD_TOPOLOGY.toml
#   2. Finds build_stages.compiler-runtime
#   3. Gets its artifact_groups: ["compiler", "core-runtime", "hip-runtime", "profiler-core"]
#   4. For each artifact_group, looks up which source_sets it needs
#   5. Collects all unique submodules from those source_sets
#   6. Downloads only those submodules: llvm-project, HIPIFY, spirv-llvm-translator
# Result: Downloads 3 submodules instead of 20+
```

**Concrete example:**

```
Full download (no source sets):
  rocm-libraries/      ← 2.5 GB
  compiler/amd-llvm/   ← 1.8 GB
  compiler/hipify/     ← 50 MB
  rocgdb/              ← 300 MB
  ... (17 more)
  Total: ~8 GB

Partial download (compiler source set only):
  compiler/amd-llvm/   ← 1.8 GB
  compiler/hipify/     ← 50 MB
  compiler/spirv-llvm-translator/ ← 100 MB
  Total: ~2 GB (saved 6 GB and download time!)
```

**Fundamental concepts:**

1. **Source sets are labels for groups of submodules** - like folders organizing files
2. **Submodule names come from .gitmodules** - this is native git, not TheRock-specific
3. **Source sets enable partial checkouts** - download only what you need
4. **This is purely about git operations** - has nothing to do with how code is compiled

### Level 2: Build Stages

#### The Problem: Building Everything Takes Too Long

**Scenario:** You have a factory making cars. Making one complete car takes 8 hours. If you make them one at a time, 10 cars = 80 hours.

**Solution:** Parallelize! Have different teams work on different parts simultaneously:
- Team A: Makes engines (2 hours)
- Team B: Makes chassis (3 hours) - can start while Team A works
- Team C: Makes interiors (2 hours) - can start after chassis is done

Total time: Not 80 hours, maybe 15 hours with good parallelization!

**Same problem in software:** Building all of ROCm takes 4-8 hours. Building everything sequentially would take forever in CI. Build stages let us run multiple builds in parallel.

#### What is CI? (Quick Background)

**CI = Continuous Integration** - automated building and testing

When developers push code to GitHub, CI automatically:
1. Checks out the code
2. Builds it
3. Runs tests
4. Reports results

**CI Job** = One build task running on one machine (like one factory team)

**Build Stage in TheRock** = A group of components that build together in one CI job

#### Build Stages Example

**Real example from BUILD_TOPOLOGY.toml:**

```toml
# Build stages define CI job boundaries
[build_stages.foundation]
description = "Foundation - critical path dependencies"
artifact_groups = ["third-party-sysdeps", "base"]
# No dependencies - this runs first

[build_stages.compiler-runtime]
description = "Compiler, runtimes, and core profiling"
artifact_groups = [
    "compiler",
    "core-runtime",
    "hip-runtime",
    "profiler-core"
]
# Dependencies: Needs foundation (see artifact_groups dependencies below)

[build_stages.math-libs]
description = "Math and ML libraries per architecture"
artifact_groups = ["math-libs", "ml-libs"]
type = "per-arch"  # Special: run once per GPU family
# Dependencies: Needs compiler-runtime (see artifact_groups dependencies below)

[build_stages.comm-libs]
description = "Communication libraries per architecture"
artifact_groups = ["comm-libs"]
type = "per-arch"
# Dependencies: Needs compiler-runtime (can run parallel with math-libs)
```

**Wait - where are the dependencies?**

Notice there's no `depends_on` field in build_stages! That's because **dependencies are inferred from artifact_groups**.

Here are the actual artifact_groups definitions (also in BUILD_TOPOLOGY.toml):

```toml
# Artifact groups define dependencies
[artifact_groups.third-party-sysdeps]
description = "Third-party system libraries"
type = "generic"
# No artifact_group_deps - this is a foundation piece

[artifact_groups.base]
description = "Base ROCm infrastructure"
type = "generic"
artifact_group_deps = []  # Empty list means no dependencies

[artifact_groups.compiler]
description = "AMD LLVM toolchain"
type = "generic"
artifact_group_deps = ["third-party-sysdeps"]  # Needs sysdeps first!
source_sets = ["compilers"]

[artifact_groups.core-runtime]
description = "Core runtime (ROCR-Runtime)"
type = "generic"
artifact_group_deps = ["base", "third-party-sysdeps"]  # Needs base AND sysdeps
source_sets = ["rocm-systems"]

[artifact_groups.hip-runtime]
description = "HIP runtime"
type = "generic"
artifact_group_deps = ["compiler", "core-runtime"]  # Needs compiler AND runtime!
source_sets = ["rocm-systems"]

[artifact_groups.math-libs]
description = "Math libraries (BLAS, FFT, etc.)"
type = "per-arch"
artifact_group_deps = ["hip-runtime"]  # Needs HIP to compile GPU kernels
source_sets = ["rocm-libraries", "math-libs"]
```

**How dependencies are computed:**

```
Build stage: compiler-runtime
  Contains artifact_groups: ["compiler", "core-runtime", "hip-runtime", ...]

Dependency computation:
  1. compiler depends on: ["third-party-sysdeps"]
  2. core-runtime depends on: ["base", "third-party-sysdeps"]
  3. hip-runtime depends on: ["compiler", "core-runtime"]

  Combined: This stage needs ["base", "third-party-sysdeps"] first
            (which are in the foundation stage)

Therefore: compiler-runtime stage must wait for foundation stage to complete!
```

**The complete picture:**

```
foundation stage builds:
  - third-party-sysdeps (no deps)
  - base (no deps)
  └─> Can run first!

compiler-runtime stage builds:
  - compiler (needs: third-party-sysdeps from foundation)
  - core-runtime (needs: base, third-party-sysdeps from foundation)
  - hip-runtime (needs: compiler, core-runtime from same stage)
  └─> Must wait for foundation to finish and upload artifacts

math-libs stage builds:
  - math-libs (needs: hip-runtime from compiler-runtime)
  - ml-libs (needs: math-libs, hip-runtime)
  └─> Must wait for compiler-runtime to finish and upload artifacts
```

#### What "type = per-arch" Means

**Background:** ROCm supports different GPU families:
- `gfx94X-dcgpu` = MI300 data center GPUs
- `gfx1100` = Radeon RX 7000 consumer GPUs
- `gfx950-dcgpu` = MI340 data center GPUs

**Problem:** Math libraries have GPU-specific kernel code. The rocBLAS library compiled for MI300 won't work on RX 7000.

**Solution:** Build separately for each GPU family.

**type values:**

- **`type = "generic"`** (or no type specified) = Build once, works for all GPUs
  - Example: Compiler (all GPUs use the same compiler)

- **`type = "per-arch"`** = Build separately for each GPU family
  - Example: rocBLAS (has GPU-specific optimized kernels)

**What happens in CI with per-arch:**

```
GitHub Actions nightly workflow specifies which GPU families to build
(configured in build_tools/github_actions/amdgpu_family_matrix.py):

amdgpu_family_info_matrix_nightly = {
    "gfx94x": {..., "family": "gfx94X-dcgpu"},
    "gfx110x": {..., "family": "gfx1100"},
    "gfx950": {..., "family": "gfx950-dcgpu"},
}

BUILD_TOPOLOGY.toml says:
  [build_stages.math-libs]
  type = "per-arch"  ← CMake uses this INSIDE build jobs (NOT GitHub Actions!)

GitHub Actions configure_ci.py script:
  ⚠️  Does NOT read BUILD_TOPOLOGY.toml!
  ⚠️  Does NOT see type = "per-arch"!

  What it actually does:
  1. Reads GPU families ONLY from amdgpu_family_matrix.py
  2. Creates one job per GPU family
  3. That's it - no knowledge of build stages or type="per-arch"!

Result: 3 parallel CI jobs created:
  Job 1: cmake -DTHEROCK_AMDGPU_FAMILIES=gfx94X-dcgpu (Machine A)
  Job 2: cmake -DTHEROCK_AMDGPU_FAMILIES=gfx1100 (Machine B, runs simultaneously!)
  Job 3: cmake -DTHEROCK_AMDGPU_FAMILIES=gfx950-dcgpu (Machine C, runs simultaneously!)

Instead of 12 hours sequential, takes 4 hours parallel!
```

**CRITICAL UNDERSTANDING: Two Different Workflows**

The above describes **ci_nightly** (monolithic workflow). There's also **multi_arch** (sharded workflow) which works differently:

**Example assumes building for 3 GPU families: gfx94X-dcgpu, gfx1100, gfx950-dcgpu**

| Aspect | Monolithic (ci_nightly) | Sharded (multi_arch) |
|--------|------------------------|----------------------|
| **Job Creation** | configure_ci.py reads amdgpu_family_matrix.py ONLY | Workflow file manually defines stage jobs |
| **Knows about stages?** | NO - just creates GPU-based jobs | YES - foundation, compiler-runtime, math-libs, etc. |
| **Generic components** | Built N times (once per GPU family) <br> **Example: 3 times for 3 GPUs** | Built ONCE (foundation + compiler-runtime jobs) |
| **Per-arch components** | Built N times (once per GPU family) <br> **Example: 3 times for 3 GPUs** | Built N times (math-libs with GPU matrix) <br> **Example: 3 times for 3 GPUs** |
| **Artifact sharing** | NO - each job is independent | YES - S3 via artifact_manager.py |
| **Build time** | Example: 4 hours (all parallel) | Example: 5.5 hours (sequential stages) |
| **Compute waste** | High (compiler built once per GPU) | Low (compiler built once total) |

**The key difference:**
- **Monolithic**: configure_ci.py creates simple GPU matrix, each job builds everything
  - If building 5 GPU families → compiler built 5 times
- **Sharded**: .github/workflows file has hardcoded stage structure, uses BUILD_TOPOLOGY.toml inside jobs
  - No matter how many GPU families → compiler built 1 time

See "Are Generic Builds Repeated in Each Matrix Job?" section below for detailed explanation with actual workflow code.

**Remember:**
- **amdgpu_family_matrix.py** defines which GPU families to build (configure_ci.py reads THIS)
- **BUILD_TOPOLOGY.toml** says `type = "per-arch"` (CMake uses this INSIDE each job, NOT for job creation)
- **GitHub Actions** creates parallel jobs based ONLY on amdgpu_family_matrix.py (monolithic) OR manually coded stages (sharded)
- **CMake** reads BUILD_TOPOLOGY.toml to decide what to build for the given `-DTHEROCK_AMDGPU_FAMILIES`

#### How CI Uses Build Stages

**IMPORTANT: Build stages are NOT used for job matrix creation!**

configure_ci.py creates a simple GPU family matrix. Build stages are used LATER inside each job.

**Actual workflow from .github/workflows/setup.yml:**

```yaml
# .github/workflows/setup.yml (simplified)

jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - name: Configure CI options
        run: |
          # ⚠️  Does NOT pass --topology flag!
          # ⚠️  Does NOT read BUILD_TOPOLOGY.toml!
          python build_tools/github_actions/configure_ci.py \
            --trigger-type presubmit
    outputs:
      # Simple GPU family matrix (no stages!):
      # [
      #   {family: "gfx94X-dcgpu", variant: "release"},
      #   {family: "gfx1100", variant: "release"},
      # ]
      linux_variants: ${{ steps.configure.outputs.linux_variants }}

  # Each GPU family becomes ONE build job:
  build:
    needs: setup
    strategy:
      matrix: ${{ fromJson(needs.configure.outputs.matrix) }}
    runs-on: azure-scale-set
    steps:
      - name: Build stage ${{ matrix.stage }} for ${{ matrix.gpu }}
        run: |
          cmake -B build -DTHEROCK_AMDGPU_FAMILIES=${{ matrix.gpu }}
          ninja -C build
```

**Visual flow:**

```
Time 0:00
┌────────────────┐
│ foundation     │  Job 1: Builds zlib, cmake, etc.
│ (generic)      │  Duration: 30 min
└───────┬────────┘
        │ Wait for foundation to upload artifacts...
        │
Time 0:30
        ↓
┌────────────────────┐
│ compiler-runtime   │  Job 2: Downloads foundation artifacts
│ (generic)          │  Builds compiler, HIP runtime
└───────┬────────────┘  Duration: 2 hours
        │ Wait for compiler-runtime to upload artifacts...
        │
Time 2:30
        ↓
        ├──────────────┬──────────────┬──────────────┬──────────────┐
        ↓              ↓              ↓              ↓              ↓
┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ math-libs    │ │ math-libs    │ │ math-libs    │ │ comm-libs    │ │ comm-libs    │
│ gfx94X       │ │ gfx1100      │ │ gfx950       │ │ gfx94X       │ │ gfx1100      │
└──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘
  Job 3           Job 4            Job 5            Job 6            Job 7
  (all run in parallel - all download compiler-runtime artifacts)
  Duration: 3 hours each

Time 5:30 - All done!

Total time: 5.5 hours
Without parallelization: 0.5 + 2 + (3×5) = 17.5 hours
Speedup: 3.2x
```

#### Concrete Example: What Runs Where

**CRITICAL: WHERE does the cmake command run? How does it know what to build?**

```bash
# CI Job for gfx94X (runs on one Azure machine)

# STEP 1: WHERE - Run from TheRock monorepo root
cd /home/runner/work/TheRock/TheRock  # ← Monorepo root directory

# STEP 2: HOW - CMake flags determine what gets built
cmake -B build \
  -DTHEROCK_AMDGPU_FAMILIES=gfx94X-dcgpu \
  \
  # Foundation & compiler components (ENABLED BY DEFAULT - no need to specify):
  # -DTHEROCK_ENABLE_BASE=ON              # ← Already ON by default
  # -DTHEROCK_ENABLE_THIRD_PARTY_SYSDEPS=ON  # ← Already ON by default
  # -DTHEROCK_ENABLE_COMPILER=ON          # ← Already ON by default
  # -DTHEROCK_ENABLE_DEVICE_LIBS=ON       # ← Already ON by default
  # -DTHEROCK_ENABLE_CORE_RUNTIME=ON      # ← Already ON by default
  # -DTHEROCK_ENABLE_HIP_RUNTIME=ON       # ← Already ON by default
  \
  # Math/ML library components (DISABLED BY DEFAULT - must enable explicitly):
  -DTHEROCK_ENABLE_MATH_LIBS=ON \        # ← Enables math-libs group
  -DTHEROCK_ENABLE_BLAS=ON \             # ← Specifically enable rocBLAS
  -DTHEROCK_ENABLE_FFT=ON \              # ← Specifically enable rocFFT
  -DTHEROCK_ENABLE_RAND=ON \             # ← Specifically enable rocRAND
  -DTHEROCK_ENABLE_SOLVER=ON \           # ← Specifically enable rocSOLVER
  -DTHEROCK_ENABLE_ML_LIBS=ON \          # ← Enables ml-libs group
  -DTHEROCK_ENABLE_MIOPEN=ON             # ← Specifically enable MIOpen

# What happens during CMake configure (in plain English):
# CMake is a build system generator - it creates the instructions for how to compile code.
#
# When you run "cmake -B build -DTHEROCK_ENABLE_BLAS=ON":
#   1. CMake reads BUILD_TOPOLOGY.toml (via Python script)
#   2. Sees that "blas" artifact is in the "math-libs" group
#   3. Sees THEROCK_ENABLE_BLAS=ON flag (user wants rocBLAS)
#   4. Tells the build system: "Include rocBLAS source code in the build"
#   5. Repeats for FFT, RAND, SOLVER, MIOpen (all enabled with =ON flags)
#
# IMPORTANT: Foundation and compiler components are ENABLED BY DEFAULT.
# You only need to explicitly enable optional components (math-libs, ml-libs, etc.)
#
# Think of it like checking boxes on a form:
#   ☑ Build compiler (default - always checked)
#   ☑ Build HIP runtime (default - always checked)
#   ☑ Build rocBLAS (optional - YOU enabled this)
#   ☑ Build rocFFT (optional - YOU enabled this)
#   ☑ Build rocRAND (optional - YOU enabled this)
#   ☑ Build rocSOLVER (optional - YOU enabled this)
#   ☑ Build MIOpen (optional - YOU enabled this)
#   ☐ Build rocSPARSE (optional - you didn't enable, so skip)

# STEP 3: Build ALL enabled components in one command
ninja -C build
# "ninja" is the actual compiler tool that builds C++ code
# One command builds everything that was enabled above

# How does one command build multiple projects?
# TheRock's structure (simplified):
#
# TheRock/                              ← Monorepo root (where you run cmake)
#   CMakeLists.txt                      ← Main build file says "if BLAS enabled, build it"
#   ├── math-libs/BLAS/CMakeLists.txt   ← This says "build rocBLAS from rocm-libraries/"
#   ├── math-libs/FFT/CMakeLists.txt    ← This says "build rocFFT from rocm-libraries/"
#   └── rocm-libraries/                 ← Git submodule with actual source code
#       ├── rocBLAS/                    ← Real C++ code for BLAS
#       ├── rocFFT/                     ← Real C++ code for FFT
#       └── rocRAND/                    ← Real C++ code for RAND
#
# When ninja runs:
#   1. Compiles rocm-libraries/rocBLAS/*.cpp → librocblas.so
#   2. Compiles rocm-libraries/rocFFT/*.cpp → librocfft.so
#   3. Compiles rocm-libraries/rocRAND/*.cpp → librocrand.so
#   4. Compiles MIOpen/*.cpp → libMIOpen.so
#
# All at once, using all CPU cores

# Each component builds into its own subdirectory:
build/
├── math-libs/BLAS/rocBLAS/build/    ← rocBLAS builds here
├── math-libs/FFT/rocFFT/build/      ← rocFFT builds here
├── math-libs/RAND/rocRAND/build/    ← rocRAND builds here
└── ml-libs/MIOpen/MIOpen/build/     ← MIOpen builds here

# STEP 4: Package artifacts
ninja -C build artifacts

# Produces (all in build/artifacts/):
- therock-blas-linux-gfx94X-dcgpu.tar.xz    ← from math-libs/BLAS/rocBLAS/stage/
- therock-fft-linux-gfx94X-dcgpu.tar.xz     ← from math-libs/FFT/rocFFT/stage/
- therock-rand-linux-gfx94X-dcgpu.tar.xz    ← from math-libs/RAND/rocRAND/stage/
- therock-miopen-linux-gfx94X-dcgpu.tar.xz  ← from ml-libs/MIOpen/MIOpen/stage/
```

**Key Insights:**

1. **WHERE**: Always run from TheRock monorepo root (top-level directory)
2. **HOW**: CMake flags (`THEROCK_ENABLE_*`) determine which components build
3. **WHAT**: TheRock's top-level CMakeLists.txt includes component subdirectories based on enabled flags
4. **STRUCTURE**: Each component builds in its own `build/.../build/` subdirectory
5. **OUTPUT**: All `.tar.xz` files go to `build/artifacts/` regardless of where component built

**Simultaneously, another CI job for gfx1100:**

```bash
# Different Azure machine, same process, different GPU flag

cd /home/runner/work/TheRock/TheRock  # ← Still monorepo root

cmake -B build \
  -DTHEROCK_AMDGPU_FAMILIES=gfx1100 \  # ← Only difference: GPU family
  # (Same enable flags as gfx94X job above - foundation/compiler default ON)
  -DTHEROCK_ENABLE_MATH_LIBS=ON \
  -DTHEROCK_ENABLE_BLAS=ON \
  -DTHEROCK_ENABLE_FFT=ON \
  -DTHEROCK_ENABLE_RAND=ON \
  -DTHEROCK_ENABLE_SOLVER=ON \
  -DTHEROCK_ENABLE_ML_LIBS=ON \
  -DTHEROCK_ENABLE_MIOPEN=ON

ninja -C build
ninja -C build artifacts

# Produces (same components, different GPU optimization):
- therock-blas-linux-gfx1100.tar.xz     ← rocBLAS optimized for RX 7000
- therock-fft-linux-gfx1100.tar.xz      ← rocFFT optimized for RX 7000
- therock-rand-linux-gfx1100.tar.xz     ← rocRAND optimized for RX 7000
- therock-miopen-linux-gfx1100.tar.xz   ← MIOpen optimized for RX 7000
```

**The magic:** Both jobs run at the same time on different machines, each building the same components but with different GPU optimizations!

#### Fundamental Concepts

1. **Build stage = CI job boundary** - defines what builds together
2. **Stages enable parallelization** - multiple stages run simultaneously if independent
3. **type = "per-arch" multiplies jobs** - one job per GPU family
4. **Build stages don't affect local builds** - only relevant for CI
5. **Dependencies between stages are computed automatically** - from artifact_groups dependencies

**Analogy:**

```
Build stages = Assembly line stations

Station 1 (foundation):
  Makes basic parts everyone needs (wheels, bolts)

Station 2 (compiler-runtime):
  Makes engines (needs Station 1's parts)

Station 3a, 3b, 3c (math-libs × 3 GPUs):
  Make 3 different car models simultaneously
  All need Station 2's engines
  All can work in parallel!
```

### Level 3: Artifact Groups

**What they are:** Logical groupings of related artifacts with shared dependencies

**Why they exist:** Components often have common dependencies and should be built together. Artifact groups express these relationships.

#### How BUILD_TOPOLOGY.toml Converts to CMake Flags

**This is the critical connection between TOML configuration and what you type on the command line.**

Let's trace the complete flow from BUILD_TOPOLOGY.toml to CMake flags:

**STEP 1: Define the artifact group in BUILD_TOPOLOGY.toml**

```toml
[artifact_groups.math-libs]
description = "Math libraries (BLAS, FFT, etc.)"
type = "per-arch"  # Built separately for each GPU family
source_sets = ["rocm-libraries", "math-libs"]  # Which git repos needed
artifact_group_deps = ["hip-runtime"]  # Depends on HIP being built first

[artifact_groups.ml-libs]
description = "ML libraries (MIOpen, etc.)"
type = "per-arch"
artifact_group_deps = ["math-libs", "hip-runtime"]  # Depends on math libs + HIP
```

**STEP 2: Define individual artifacts that belong to these groups**

```toml
# Individual artifacts in the math-libs group
[artifacts.blas]
artifact_group = "math-libs"    # ← Belongs to math-libs group
type = "target-specific"
feature_name = "BLAS"           # ← Creates THEROCK_ENABLE_BLAS
feature_group = "MATH_LIBS"     # ← Belongs to MATH_LIBS feature group

[artifacts.fft]
artifact_group = "math-libs"
type = "target-specific"
feature_name = "FFT"            # ← Creates THEROCK_ENABLE_FFT
feature_group = "MATH_LIBS"

[artifacts.rand]
artifact_group = "math-libs"
type = "target-specific"
feature_name = "RAND"           # ← Creates THEROCK_ENABLE_RAND
feature_group = "MATH_LIBS"

[artifacts.solver]
artifact_group = "math-libs"
type = "target-specific"
feature_name = "SOLVER"         # ← Creates THEROCK_ENABLE_SOLVER
feature_group = "MATH_LIBS"

# Individual artifact in the ml-libs group
[artifacts.miopen]
artifact_group = "ml-libs"
type = "target-specific"
feature_name = "MIOPEN"         # ← Creates THEROCK_ENABLE_MIOPEN
feature_group = "ML_LIBS"       # ← Belongs to ML_LIBS feature group
```

**STEP 3: Python script generates CMake code (topology_to_cmake.py)**

When you run `cmake -B build`, CMake executes `topology_to_cmake.py` which reads the TOML and generates this:

```cmake
# File: build/cmake/therock_topology_generated.cmake
# (Auto-generated - DO NOT EDIT)

# Feature group for math libraries
therock_add_feature(MATH_LIBS
  DESCRIPTION "Enable all math libraries"
)

# Individual features in the MATH_LIBS group
therock_add_feature(BLAS
  GROUP MATH_LIBS              # ← Links to MATH_LIBS group
  DESCRIPTION "Enables blas"
  REQUIRES HIP_RUNTIME         # ← Dependency
)

therock_add_feature(FFT
  GROUP MATH_LIBS
  DESCRIPTION "Enables fft"
  REQUIRES HIP_RUNTIME
)

therock_add_feature(RAND
  GROUP MATH_LIBS
  DESCRIPTION "Enables rand"
  REQUIRES HIP_RUNTIME
)

therock_add_feature(SOLVER
  GROUP MATH_LIBS
  DESCRIPTION "Enables solver"
  REQUIRES HIP_RUNTIME BLAS    # ← Solver depends on BLAS too!
)

# Feature group for ML libraries
therock_add_feature(ML_LIBS
  DESCRIPTION "Enable all ML libraries"
)

therock_add_feature(MIOPEN
  GROUP ML_LIBS
  DESCRIPTION "Enables miopen"
  REQUIRES HIP_RUNTIME BLAS    # ← MIOpen needs HIP and BLAS
)
```

**STEP 4: This creates CMake cache variables you can control**

The `therock_add_feature()` function (defined in `cmake/therock_features.cmake:22`) creates ON/OFF switches:

```cmake
# What therock_add_feature() creates (in CMake cache):

# Group-level switches (OFF by default because they're groups):
THEROCK_ENABLE_MATH_LIBS=OFF   # Master switch for all math libs
THEROCK_ENABLE_ML_LIBS=OFF     # Master switch for all ML libs

# Individual component switches (also OFF by default because they're in a group):
THEROCK_ENABLE_BLAS=OFF
THEROCK_ENABLE_FFT=OFF
THEROCK_ENABLE_RAND=OFF
THEROCK_ENABLE_SOLVER=OFF
THEROCK_ENABLE_MIOPEN=OFF
```

**STEP 5: You control these with command-line flags**

Now you understand what each flag does!

**Option A: Enable entire group (easiest)**

```bash
cmake -B build \
  -DTHEROCK_AMDGPU_FAMILIES=gfx1100 \
  -DTHEROCK_ENABLE_MATH_LIBS=ON     # ← Enables ALL math libs (BLAS, FFT, RAND, SOLVER)

# Result: All artifacts in math-libs group are built:
#   ✓ rocBLAS (from artifact.blas)
#   ✓ rocFFT (from artifact.fft)
#   ✓ rocRAND (from artifact.rand)
#   ✓ rocSOLVER (from artifact.solver)
```

**Option B: Enable specific artifacts only (selective)**

```bash
cmake -B build \
  -DTHEROCK_AMDGPU_FAMILIES=gfx1100 \
  -DTHEROCK_ENABLE_BLAS=ON          # ← Only enable BLAS
  -DTHEROCK_ENABLE_FFT=ON           # ← Only enable FFT
  # (Don't enable RAND or SOLVER)

# Result: Only the ones you specified are built:
#   ✓ rocBLAS (explicitly enabled)
#   ✓ rocFFT (explicitly enabled)
#   ✗ rocRAND (not enabled, so skipped)
#   ✗ rocSOLVER (not enabled, so skipped)

# Note: HIP_RUNTIME will be enabled automatically because BLAS and FFT
# require it (dependencies are auto-enabled)
```

**Option C: Enable group but disable specific artifacts (exclude from group)**

```bash
cmake -B build \
  -DTHEROCK_AMDGPU_FAMILIES=gfx1100 \
  -DTHEROCK_ENABLE_MATH_LIBS=ON     # ← Enable the whole group
  -DTHEROCK_ENABLE_SOLVER=OFF       # ← But explicitly turn off SOLVER

# Result: All math libs EXCEPT solver:
#   ✓ rocBLAS (group enabled)
#   ✓ rocFFT (group enabled)
#   ✓ rocRAND (group enabled)
#   ✗ rocSOLVER (explicitly disabled)
```

**Option D: Enable everything (master switch)**

```bash
cmake -B build \
  -DTHEROCK_AMDGPU_FAMILIES=gfx1100 \
  -DTHEROCK_ENABLE_ALL=ON           # ← Master switch: enables ALL groups

# Result: Every optional component is built
#   ✓ All math libs (BLAS, FFT, RAND, SOLVER, SPARSE, etc.)
#   ✓ All ML libs (MIOpen, composable_kernel, etc.)
#   ✓ All comm libs (RCCL)
#   ✓ All profiling tools
```

**The Hierarchy in Action:**

```
BUILD_TOPOLOGY.toml:
  [artifact_groups.math-libs]     ← Defines the group
    └─ Contains these artifacts:
         [artifacts.blas]         ← Individual artifact
           feature_group = "MATH_LIBS"
           feature_name = "BLAS"
         [artifacts.fft]
           feature_group = "MATH_LIBS"
           feature_name = "FFT"

topology_to_cmake.py converts to:
  therock_add_feature(MATH_LIBS)  ← Group-level ON/OFF switch
  therock_add_feature(BLAS GROUP MATH_LIBS)  ← Individual ON/OFF switch
  therock_add_feature(FFT GROUP MATH_LIBS)   ← Individual ON/OFF switch

Which creates CMake cache variables:
  THEROCK_ENABLE_MATH_LIBS=ON/OFF  ← Controls all math libs at once
  THEROCK_ENABLE_BLAS=ON/OFF       ← Controls just BLAS
  THEROCK_ENABLE_FFT=ON/OFF        ← Controls just FFT

You control with command line:
  cmake -DTHEROCK_ENABLE_MATH_LIBS=ON   ← Enable all at once
  cmake -DTHEROCK_ENABLE_BLAS=ON        ← Enable just BLAS
```

**What this means:**

```
Artifact group dependencies:
  ml-libs depends on → math-libs → hip-runtime

Build order (computed automatically):
  1. hip-runtime  (no dependencies)
  2. math-libs    (depends on hip-runtime)
  3. ml-libs      (depends on math-libs and hip-runtime)
```

**Fundamental concepts:**

1. **Artifact groups are about dependencies** - what needs what
2. **type = "per-arch"** means each GPU family gets separate builds
3. **This is still metadata** - Python script uses this to compute build order

#### How Python Generates CMake Code

**The simple story: Python creates a CMake file that calls a predefined function.**

```
BUILD_TOPOLOGY.toml            Python Script              Generated CMake File
━━━━━━━━━━━━━━━━━━━          ━━━━━━━━━━━━━━━━          ━━━━━━━━━━━━━━━━━━━━━━━━
[artifacts.blas]       ──→    topology_to_cmake.py  ──→  therock_add_feature(BLAS
feature_name = "BLAS"          reads TOML                   GROUP MATH_LIBS
feature_group =                writes text file             REQUIRES HIP_RUNTIME
  "MATH_LIBS"                                             )
artifact_deps =
  ["core-hip"]
```

**What happens:**

1. **Python reads TOML** → Gets `feature_name = "BLAS"`, `feature_group = "MATH_LIBS"`, etc.

2. **Python writes a CMake file** (`build/cmake/therock_topology_generated.cmake`) containing function calls:
   ```cmake
   therock_add_feature(BLAS GROUP MATH_LIBS REQUIRES HIP_RUNTIME)
   therock_add_feature(FFT GROUP MATH_LIBS REQUIRES HIP_RUNTIME)
   therock_add_feature(SOLVER GROUP MATH_LIBS REQUIRES HIP_RUNTIME BLAS)
   ```

3. **CMake includes this generated file** and executes the function calls

4. **The `therock_add_feature()` function is PREDEFINED** (in `cmake/therock_features.cmake:22`)
   - Python does NOT create this function
   - Python just CALLS it with different parameters
   - The function creates `THEROCK_ENABLE_BLAS` cache variables that you can override with `-D` flags

**Analogy:**

```
Python = Secretary writing form letters
  - Has a template: "therock_add_feature(NAME, GROUP, REQUIRES)"
  - Fills in the blanks from TOML data
  - Saves the letters to a file

CMake = Boss reading the letters
  - Opens the file of letters
  - Executes each instruction
  - Creates the actual ON/OFF switches you control
```

**The key insight:** `therock_add_feature()` is like a pre-existing rubber stamp. Python doesn't create the stamp, it just decides when to use it and what text to stamp.

### Level 4: Artifacts

**What they are:** Individual `.tar.xz` files - the actual build outputs

**Why they exist:** This is the fundamental packaging unit. One artifact = one `.tar.xz` file.

**Example:**

```toml
[artifacts.blas]
artifact_group = "math-libs"           # Belongs to math-libs group
type = "target-specific"               # Each GPU family gets separate file
artifact_deps = ["core-hip"]           # Needs HIP runtime
feature_name = "BLAS"                  # Creates THEROCK_ENABLE_BLAS variable
feature_group = "MATH_LIBS"            # Groups with other math lib options
split_databases = ["rocblas"]          # Special handling for kernel databases

[artifacts.compiler]
artifact_group = "compiler"
type = "target-neutral"                # One build for all GPU families
feature_name = "COMPILER"
feature_group = "COMPILER"
```

**What this creates:**

```cmake
# Generated by topology_to_cmake.py in build/cmake/therock_topology.cmake:

# What is therock_add_feature()?
# This is a CMake function defined in cmake/therock_features.cmake (line 22)
# It creates a user-configurable ON/OFF switch for each component.
#
# When called, it creates these CMake variables:
#   THEROCK_ENABLE_{name} = ON/OFF (cache variable user can change)
#   THEROCK_REQUIRES_{name} = list of dependencies
#   THEROCK_PLATFORM_DISABLED_{name} = platforms where disabled
#
# File: cmake/therock_features.cmake
# Line 22: function(therock_add_feature feature_name)
# Line 82: set(THEROCK_ENABLE_${feature_name} ${_default_enabled} CACHE BOOL ...)

# Feature declaration (creates cache variable)
therock_add_feature(BLAS
  GROUP MATH_LIBS
  DESCRIPTION "Enables blas"
  REQUIRES HIP_RUNTIME  # Translated from artifact_deps
)

# This specific call creates:
#   THEROCK_ENABLE_BLAS = ON/OFF  (user can toggle with -DTHEROCK_ENABLE_BLAS=OFF)
#   THEROCK_REQUIRES_BLAS = HIP_RUNTIME  (CMake will check HIP_RUNTIME is enabled)

# CMake target
add_custom_target(artifact-blas
  COMMENT "Building artifact blas"
)

# Metadata for packaging
set(THEROCK_ARTIFACT_TYPE_blas "target-specific")
set(THEROCK_ARTIFACT_GROUP_blas "math-libs")
set(THEROCK_ARTIFACT_SPLIT_DATABASES_blas "rocblas")
```

**Fundamental concepts:**

1. **Artifact = one .tar.xz file** - this is the concrete output
2. **type values:**
   - `target-neutral`: Built once for all GPUs (e.g., compiler, headers)
   - `target-specific`: Built separately per GPU (e.g., rocBLAS kernels)
3. **Each artifact creates:**
   - A CMake feature flag (`THEROCK_ENABLE_BLAS`)
   - A CMake target (`artifact-blas`)
   - Metadata variables for packaging

### Putting It All Together: rocBLAS Example

Let's trace rocBLAS through all four levels:

```toml
# LEVEL 1: Source Set
[source_sets.rocm-libraries]
submodules = ["rocm-libraries"]  # Git repo containing rocBLAS source

# LEVEL 2: Build Stage
[build_stages.math-libs]
artifact_groups = ["math-libs"]  # This stage builds math libraries
type = "per-arch"                # Run once per GPU family

# LEVEL 3: Artifact Group
[artifact_groups.math-libs]
type = "per-arch"
artifact_group_deps = ["hip-runtime"]  # Needs HIP first

# LEVEL 4: Artifact
[artifacts.blas]
artifact_group = "math-libs"
type = "target-specific"
artifact_deps = ["core-hip"]
```

**What happens when you build:**

```bash
# REALISTIC CI EXAMPLE - What actually happens in CI

# 1. CMake configure - Python reads BUILD_TOPOLOGY.toml
cmake -B build \
  -DTHEROCK_ENABLE_BLAS=ON \
  -DTHEROCK_ENABLE_FFT=ON \
  -DTHEROCK_ENABLE_SOLVER=ON \
  -DTHEROCK_ENABLE_RAND=ON \
  -DTHEROCK_AMDGPU_FAMILIES=gfx94X-dcgpu

# Generated CMake code creates targets for all enabled components:
#   THEROCK_ENABLE_BLAS = ON    → artifact-blas target
#   THEROCK_ENABLE_FFT = ON     → artifact-fft target
#   THEROCK_ENABLE_SOLVER = ON  → artifact-solver target
#   THEROCK_ENABLE_RAND = ON    → artifact-rand target

# 2. Build ALL enabled components
ninja -C build
# OR: ninja -C build therock-artifacts (to also create .tar.xz files)

# What happens with parallelism (3 levels):
#
# LEVEL 1: Component-level parallelism
#   ninja builds multiple components at the same time:
#   - Thread group 1-16:  Compiling rocBLAS  (blas/src/*.cpp)
#   - Thread group 17-32: Compiling rocFFT   (fft/src/*.cpp)
#   - Thread group 33-48: Compiling rocSOLVER (solver/src/*.cpp)
#   - Thread group 49-64: Compiling rocRAND  (rand/src/*.cpp)
#
#   All 4 components build IN PARALLEL (if independent)
#   If solver depends on blas, ninja builds blas first, then solver
#
# LEVEL 2: File-level parallelism (within each component)
#   Within rocBLAS alone:
#   - Thread 1:  Compiling rocblas_axpy.cpp
#   - Thread 2:  Compiling rocblas_gemv.cpp
#   - Thread 3:  Compiling rocblas_gemm.cpp
#   - ... (all .cpp files compile in parallel using all available CPU cores)
#
# LEVEL 3: Machine-level parallelism (CI jobs)
#   This whole build is happening on ONE machine for gfx94X
#   Simultaneously, another machine is doing the same for gfx1100
#   (See "Are Generic Builds Repeated?" section above)

# Result after ~3 hours on 64-core machine:
#   build/artifacts/therock-blas-linux-gfx94X-dcgpu.tar.xz
#   build/artifacts/therock-fft-linux-gfx94X-dcgpu.tar.xz
#   build/artifacts/therock-solver-linux-gfx94X-dcgpu.tar.xz
#   build/artifacts/therock-rand-linux-gfx94X-dcgpu.tar.xz

# 3. CI uploads ALL artifacts to S3
#   s3://therock-ci-artifacts/{run_id}-linux/therock-blas-linux-gfx94X-dcgpu.tar.xz
#   s3://therock-ci-artifacts/{run_id}-linux/therock-fft-linux-gfx94X-dcgpu.tar.xz
#   s3://therock-ci-artifacts/{run_id}-linux/therock-solver-linux-gfx94X-dcgpu.tar.xz
#   s3://therock-ci-artifacts/{run_id}-linux/therock-rand-linux-gfx94X-dcgpu.tar.xz
```

**CRITICAL: Understanding the 3 Levels of Parallelism**

```
┌─────────────────────────────────────────────────────────────────┐
│ ONE CI Build (1 Machine, 64 CPU cores, builds for gfx94X)      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│ Component-Level Parallelism:                                    │
│ ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│ │   rocBLAS   │  │   rocFFT    │  │  rocSOLVER  │  ← All build│
│ │             │  │             │  │             │    at same   │
│ │ Uses threads│  │ Uses threads│  │ Uses threads│    time      │
│ │    1-16     │  │   17-32     │  │   33-48     │             │
│ └─────────────┘  └─────────────┘  └─────────────┘             │
│         ↓                ↓                ↓                     │
│ File-Level Parallelism (within each component):                │
│  Thread 1: axpy.cpp    Thread 17: fft.cpp    Thread 33: lu.cpp│
│  Thread 2: gemv.cpp    Thread 18: ifft.cpp   Thread 34: qr.cpp│
│  Thread 3: gemm.cpp    Thread 19: plan.cpp   Thread 35: sv.cpp│
│  ... all .cpp files compile simultaneously                     │
│                                                                 │
│ Output: 4 .tar.xz files (all optimized for gfx94X)             │
└─────────────────────────────────────────────────────────────────┘

              ┌──── Machine-Level Parallelism ────┐

┌─────────────────────────────────────────────────────────────────┐
│ ANOTHER CI Build (Different Machine, builds for gfx1100)       │
│ Same process, same 4 components, different GPU optimization    │
│ Output: 4 .tar.xz files (all optimized for gfx1100)            │
└─────────────────────────────────────────────────────────────────┘
```

**To answer your question directly:**

- NO, it's not 1 thread building blas while others are idle
- YES, ninja builds multiple components in parallel (blas, fft, solver, rand)
- YES, within each component, multiple threads compile different .cpp files
- In CI, we enable MANY components (not just blas), so ninja has lots to parallelize
- ONE machine builds multiple components in parallel using all CPU cores
- MULTIPLE machines build for different GPUs simultaneously (machine-level parallelism)
```

### Summary: The Four Levels in Simple Terms

| Level | What | Example | Why |
|---|---|---|---|
| **Source Set** | Git submodules to clone | `rocm-libraries` | Partial checkouts for CI |
| **Build Stage** | CI job boundary | `math-libs` | Parallelize builds |
| **Artifact Group** | Logical grouping with deps | `math-libs` group | Express dependencies |
| **Artifact** | Actual .tar.xz file | `blas` artifact | Fundamental packaging unit |

**The key insight:**

- **Source sets** → About **git repositories**
- **Build stages** → About **CI/CD parallelization**
- **Artifact groups** → About **dependency relationships**
- **Artifacts** → About **packaging outputs** (.tar.xz files)

These are different dimensions of organization, not a strict hierarchy. The same artifact (like `blas`) participates in all four levels for different reasons.

---

## Part 3: Stage 1 - Building Portable Artifacts

### What Are "Portable Artifacts"?

A portable artifact is a `.tar.xz` file containing everything needed for one component of ROCm.

**Example:** `therock-blas-linux-gfx94X-dcgpu.tar.xz`

Extract it and you get:

**CRITICAL: The Complete Build-to-Package Flow**

Before showing the final packaged contents, let's understand the complete flow from raw source code to final `.tar.xz` file.

**Step-by-Step: How rocBLAS Goes from Source Code to .tar.xz Package**

```
STEP 1: Source Code Location (Git Submodule)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Location: rocm-libraries/rocBLAS/

This is where the raw C++ source code lives:
  rocm-libraries/rocBLAS/library/src/blas1/rocblas_axpy.cpp
  rocm-libraries/rocBLAS/library/src/blas2/rocblas_gemv.cpp
  rocm-libraries/rocBLAS/library/src/blas3/rocblas_gemm.cpp
  rocm-libraries/rocBLAS/CMakeLists.txt

Git submodule definition (.gitmodules):
  [submodule "rocm-libraries"]
    path = rocm-libraries
    url = https://github.com/ROCm/rocm-libraries

This is the SOURCE - uncompiled C++ code

      ↓ cmake configure + ninja build

STEP 2: Build Directory (CMake Compilation)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Location: build/math-libs/BLAS/rocBLAS/build/

CMake command that creates this:
  cmake -B build/math-libs/BLAS/rocBLAS/build \
        -S rocm-libraries/rocBLAS \
        -DCMAKE_INSTALL_PREFIX=build/math-libs/BLAS/rocBLAS/stage \
        -DAMDGPU_TARGETS=gfx942

What happens here:
  • C++ source files are compiled to object files (.o)
  • Object files are linked into librocblas.so.4.0.0
  • Kernel databases are generated (TensileLibrary_gfx942.dat)
  • Test binaries are built (rocblas-bench, rocblas-test)

Files in build/:
  build/math-libs/BLAS/rocBLAS/build/library/src/librocblas.so.4.0.0
  build/math-libs/BLAS/rocBLAS/build/library/src/blas1/CMakeFiles/rocblas_axpy.o
  build/math-libs/BLAS/rocBLAS/build/clients/benchmarks/rocblas-bench

This is the BUILD OUTPUT - compiled binaries, but not yet organized for installation

      ↓ ninja install (copies files to install prefix)

STEP 3: Stage Directory (CMake Install Tree)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Location: build/math-libs/BLAS/rocBLAS/stage/

CMake command that populates this:
  ninja -C build/math-libs/BLAS/rocBLAS/build install

What happens:
  • CMake copies built files to CMAKE_INSTALL_PREFIX location
  • Files are organized in standard FHS layout (lib/, bin/, include/)
  • This is the "install tree" - ready to be deployed to /opt/rocm/

Files in stage/:
  build/math-libs/BLAS/rocBLAS/stage/lib/librocblas.so.4.0.0
  build/math-libs/BLAS/rocBLAS/stage/lib/librocblas.so.4 → librocblas.so.4.0.0
  build/math-libs/BLAS/rocBLAS/stage/lib/rocblas/library/TensileLibrary_gfx942.dat
  build/math-libs/BLAS/rocBLAS/stage/bin/rocblas-bench
  build/math-libs/BLAS/rocBLAS/stage/include/rocblas/rocblas.h
  build/math-libs/BLAS/rocBLAS/stage/lib/cmake/rocblas/rocblas-config.cmake

This is the INSTALL TREE - organized exactly like it will appear in /opt/rocm/

      ↓ Packaging system reads artifact-blas.toml

STEP 4: Package Selection (artifact-blas.toml)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Location: math-libs/BLAS/artifact-blas.toml

File: math-libs/BLAS/artifact-blas.toml
```toml
# This TOML file selects which files from stage/ go into the .tar.xz

# Component: Runtime library (what users need to run applications)
[components.lib."math-libs/BLAS/rocBLAS/stage"]
include = [
  "lib/librocblas.so*",              # The library itself
  "lib/rocblas/library/**",          # Kernel database files
]

# Component: Development files (what developers need to compile)
[components.dev."math-libs/BLAS/rocBLAS/stage"]
include = [
  "include/rocblas/**",              # Header files
  "lib/cmake/rocblas/**",            # CMake config files
]

# Component: Test binaries (optional, for testing)
[components.test."math-libs/BLAS/rocBLAS/stage"]
include = [
  "bin/rocblas-bench",
  "bin/rocblas-test",
]
```

What this means:
  • Path "math-libs/BLAS/rocBLAS/stage" points to the install tree
  • include = [...] patterns select files from that tree
  • Multiple [components.*] sections split files into logical groups

This is the SELECTION RULES - which files go into the final package

      ↓ ninja artifacts (runs packaging)

STEP 5: Final Package (.tar.xz)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Location: build/artifacts/therock-blas-linux-gfx94X-dcgpu.tar.xz

Command that creates this:
  ninja -C build artifacts

What happens:
  1. Reads artifact-blas.toml
  2. Collects matching files from build/math-libs/BLAS/rocBLAS/stage/
  3. Creates .tar.xz archive with selected files
  4. Names it: therock-{artifact}-{platform}-{gpu}.tar.xz

Final archive contents:
  therock-blas-linux-gfx94X-dcgpu.tar.xz contains:
    lib/librocblas.so.4.0.0
    lib/librocblas.so.4 → librocblas.so.4.0.0
    lib/rocblas/library/TensileLibrary_gfx942.dat
    include/rocblas/rocblas.h
    lib/cmake/rocblas/rocblas-config.cmake

This is the FINAL PACKAGE - ready to distribute and install
```

**Key Points:**

1. **Source** → **Build** → **Stage** → **Package** (4 separate directories)
2. **Source** = Git submodule with raw C++ code
3. **Build** = CMake compilation directory (messy, has .o files, temporary files)
4. **Stage** = Clean install tree (organized like /opt/rocm/)
5. **Package** = Selected files from stage/ bundled into .tar.xz

**Summary of the Complete Flow:**

```
rocm-libraries/rocBLAS/                       ← SOURCE (raw C++ code)
        ↓ cmake + ninja
build/math-libs/BLAS/rocBLAS/build/           ← BUILD (compiled .o, .so)
        ↓ ninja install
build/math-libs/BLAS/rocBLAS/stage/           ← STAGE (organized for /opt/rocm/)
        ↓ ninja artifacts (reads artifact-blas.toml)
build/artifacts/therock-blas-linux-*.tar.xz   ← PACKAGE (final distributable)
```

**Why "portable"?**

These artifacts are built in a special **manylinux container** that uses an old glibc version. This ensures they work on any Linux distribution from the last ~5 years.

**Why ".tar.xz"?**

- `.tar` = archive format (bundles multiple files)
- `.xz` = compression (makes it smaller)
- Together: portable, compressed, preserves permissions and symlinks

### The Build Process (Linux)

**Workflow:** `.github/workflows/build_portable_linux_artifacts.yml`

**What happens:**

```bash
# 1. Setup (runs on Azure Scale Set)
git clone https://github.com/ROCm/TheRock.git
git submodule update --init --recursive  # Get all ROCm source repos
python3 ./build_tools/fetch_sources.py   # Apply patches

# 2. Configure with CMake
cmake -B build -GNinja \
  -DCMAKE_BUILD_TYPE=Release \
  -DTHEROCK_AMDGPU_FAMILIES=gfx94X-dcgpu \
  -DTHEROCK_ENABLE_MATH_LIBS=ON \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++

# 3. Build (this takes 4-8 hours)
ninja -C build

# 4. Create component artifacts
ninja -C build artifacts

# This produces:
# build/artifacts/
# ├── therock-base-linux-gfx94X-dcgpu.tar.xz
# ├── therock-compiler-linux-gfx94X-dcgpu.tar.xz
# ├── therock-core-linux-gfx94X-dcgpu.tar.xz
# └── therock-blas-linux-gfx94X-dcgpu.tar.xz

# 5. Upload to S3
python ./build_tools/github_actions/post_build_upload.py \
  --artifact-dir build/artifacts/ \
  --s3-bucket therock-ci-artifacts \
  --s3-prefix ${RUN_ID}-linux/
```

**S3 Result:**

```
s3://therock-ci-artifacts/21440027240-linux/
├── index-gfx94X-dcgpu.html
├── therock-base-linux-gfx94X-dcgpu.tar.xz
├── therock-compiler-linux-gfx94X-dcgpu.tar.xz
├── therock-core-linux-gfx94X-dcgpu.tar.xz
├── therock-math-libs-linux-gfx94X-dcgpu.tar.xz
└── ... (one per component)
```

The `index-gfx94X-dcgpu.html` is a simple web page listing all files, making it easy to download them.

### The Build Process (Windows)

**Workflow:** `.github/workflows/build_windows_artifacts.yml`

Almost identical to Linux, except:

- Runs on Windows runners (no container needed)
- Uses Visual Studio compiler instead of Clang
- Produces `.tar.xz` files with Windows binaries

**Output:**

```
s3://therock-ci-artifacts/21440027240-windows/
├── index-gfx94X-dcgpu.html
├── therock-base-windows-gfx94X-dcgpu.tar.xz
├── therock-compiler-windows-gfx94X-dcgpu.tar.xz
└── ... (one per component)
```

### Artifact Slicing: One Build → Multiple .tar.xz Files

How does CMake split the build output into separate component artifacts?

Each component has an `artifact-*.toml` descriptor that defines **slicing rules**:

**Example:** `math-libs/BLAS/artifact-blas.toml`

```toml
# rocBLAS library files (for runtime)
[components.lib."math-libs/BLAS/rocBLAS/stage"]
include = [
  "bin/rocblas/library/**",      # Kernel library data files
  "lib/rocblas/library/**",
]

# rocBLAS development files (for compiling against rocBLAS)
[components.dev."math-libs/BLAS/rocBLAS/stage"]
# Uses defaults: include/**, lib/cmake/**

# rocBLAS test executables
[components.test."math-libs/BLAS/rocBLAS/stage"]
include = [
  "bin/rocblas-bench*",
  "bin/rocblas-test*",
  "bin/rocblas_*.yaml",
]

# rocBLAS documentation
[components.doc."math-libs/BLAS/rocBLAS/stage"]
include = [
  "**/rocblas_clients_readme.txt",
]
```

**What CMake does:**

1. Builds everything into `build/math-libs/BLAS/rocBLAS/stage/`
2. Reads `artifact-blas.toml`
3. Creates separate component directories:
   ```
   build/artifacts/components/blas/
   ├── lib/      # Runtime files only
   ├── dev/      # Headers and CMake files
   ├── test/     # Benchmarks and tests
   └── doc/      # Documentation
   ```
4. Packages each component into the final artifact:
   ```
   therock-blas-linux-gfx94X-dcgpu.tar.xz
   ```

**Why slice?**

This allows Python packaging and native packaging to pick exactly which files they need:

- Python `rocm-sdk-libraries` wheel → Gets `lib/` component only
- Python `rocm-sdk-devel` wheel → Gets `dev/` component
- Native `rocm-dev` package → Gets `dev/` component
- Native `rocm-test` package → Gets `test/` component

One build, flexible packaging.

---

## Part 4: Stage 2A - Python Packaging

### Overview

**Workflow:** `.github/workflows/build_portable_linux_python_packages.yml`

**Input:** Component artifacts from S3 (`.tar.xz` files)

**Output:** Python wheels (`.whl` files)

**Duration:** 10-20 minutes

### What is a Python Wheel?

A wheel (`.whl`) is Python's standard package format. It's basically a ZIP file with:

- Python code (`.py` files)
- Compiled extensions (`.so` or `.pyd` files)
- Package metadata (name, version, dependencies)
- Entry points (console scripts like `rocm-sdk`)

**Example wheel filename:**

```
rocm_sdk_libraries_gfx94X_dcgpu-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl
```

Breaking this down:

- `rocm_sdk_libraries_gfx94X_dcgpu` = package name
- `7.10.0a20251124` = version (nightly from Nov 24, 2025)
- `py3` = works with Python 3.x
- `none` = no ABI requirement
- `manylinux_2_28_x86_64` = Linux with glibc ≥ 2.28, x86-64 CPU

### The Four Python Packages

TheRock produces four types of Python packages:

| Package Name | Type | Contains | GPU-Specific? |
|---|---|---|---|
| **rocm** | sdist (`.tar.gz`) | Selector logic + CLI tool | No |
| **rocm-sdk-core** | wheel (`.whl`) | HIP runtime, compiler tools | No |
| **rocm-sdk-libraries-{gpu}** | wheel (`.whl`) | Math libraries (rocBLAS, etc.) | Yes |
| **rocm-sdk-devel** | wheel (`.whl`) | Headers, CMake files, symlinks | No |

**How they relate:**

```
pip install rocm[libraries,devel]
   │
   ├─ Installs: rocm (sdist)
   ├─ rocm detects GPU: gfx94X
   ├─ rocm adds dependency: rocm-sdk-core
   ├─ rocm adds dependency: rocm-sdk-libraries-gfx94X-dcgpu
   └─ rocm adds dependency: rocm-sdk-devel
```

### The Packaging Process

**Script:** `build_tools/build_python_packages.py`

**What it does:**

```bash
# 1. Download artifacts from S3
python ./build_tools/fetch_artifacts.py \
  --run-id 21440027240 \
  --artifact-group gfx94X-dcgpu \
  --output-dir /tmp/artifacts/

# This downloads all .tar.xz files to /tmp/artifacts/

# 2. Build Python packages
python ./build_tools/build_python_packages.py \
  --artifact-dir /tmp/artifacts/ \
  --dest-dir /tmp/packages/ \
  --version 7.10.0a20251124

# This creates:
# /tmp/packages/
# ├── rocm-7.10.0a20251124.tar.gz
# ├── rocm_sdk_core-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl
# ├── rocm_sdk_libraries_gfx94X_dcgpu-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl
# └── rocm_sdk_devel-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl

# 3. Upload to S3
python ./build_tools/github_actions/upload_python_packages.py \
  --source-dir /tmp/packages/ \
  --s3-bucket therock-nightly-python \
  --s3-prefix v2/gfx94X-dcgpu/
```

### Deep Dive: Building rocm-sdk-libraries Wheel

Let's trace exactly how the math libraries wheel is built.

**Step 1: Filter Artifacts**

The script uses a filter function to decide which artifacts to include:

```python
def libraries_artifact_filter(target_family: str, an: ArtifactName) -> bool:
    """Select which artifacts go into rocm-sdk-libraries wheel."""
    return (
        an.name in ["blas", "fft", "rand", "sparse", "solver", "rccl", "miopen"]
        and an.component in ["lib"]  # Runtime files only, not dev/test/doc
        and (an.target_family == target_family or an.target_family == "generic")
    )
```

**Result:** Only math library runtime files are included.

**Step 2: Extract Artifacts**

```bash
# Extract each selected .tar.xz
cd /tmp/staging/rocm_sdk_libraries_gfx94X_dcgpu/

tar -xJf /tmp/artifacts/therock-blas-linux-gfx94X-dcgpu.tar.xz
tar -xJf /tmp/artifacts/therock-fft-linux-gfx94X-dcgpu.tar.xz
# ... repeat for each math library
```

**Directory structure:**

```
/tmp/staging/rocm_sdk_libraries_gfx94X_dcgpu/
├── math-libs/BLAS/rocBLAS/stage/
│   ├── bin/rocblas/library/TensileLibrary_gfx94X.dat
│   └── lib/librocblas.so.4.0.0
├── math-libs/FFT/rocFFT/stage/
│   └── lib/librocfft.so.1.0.0
└── ... (other math libs)
```

**Step 3: Reorganize for Python**

Python packages expect a flat structure, not the nested component structure. So we reorganize:

```
/tmp/staging/rocm_sdk_libraries_gfx94X_dcgpu/
├── rocm_sdk_libraries/
│   ├── __init__.py
│   ├── bin/
│   │   └── rocblas/
│   │       └── library/
│   │           └── TensileLibrary_gfx94X.dat
│   └── lib/
│       ├── librocblas.so.4.0.0
│       ├── librocfft.so.1.0.0
│       └── ... (all math libs)
└── setup.py
```

**Step 4: Handle Symlinks**

Wheels don't support symlinks. So we:

1. **Keep only SONAME files** (the actual binaries)
2. **Remove symlinks** (they'll be recreated by Python code at runtime)

**Example:**

Before (from build):
```
lib/librocblas.so.4.0.0        # Actual file (235 MB)
lib/librocblas.so.4 → librocblas.so.4.0.0   # Symlink
lib/librocblas.so → librocblas.so.4         # Symlink
```

After (in wheel):
```
lib/librocblas.so.4.0.0        # Actual file (235 MB)
# Symlinks removed
```

When users install, Python code recreates the symlinks dynamically.

**Step 5: Patch RPATHs**

The libraries need to find each other. We patch the RPATH (runtime library search path) to use relative paths:

```bash
patchelf --set-rpath '$ORIGIN:$ORIGIN/../lib' lib/librocblas.so.4.0.0
```

Now `librocblas.so` can find dependencies in the same wheel.

**Step 6: Add Python Wrapper**

We add a `__init__.py` that helps with initialization:

```python
# rocm_sdk_libraries/__init__.py

def get_lib_dir():
    """Return absolute path to lib/ directory in this package."""
    import pathlib
    return pathlib.Path(__file__).parent / "lib"

def get_version():
    """Return package version."""
    return "7.10.0a20251124"
```

**Step 7: Create setup.py**

```python
# setup.py (simplified)

from setuptools import setup

setup(
    name="rocm-sdk-libraries-gfx94X-dcgpu",
    version="7.10.0a20251124",
    packages=["rocm_sdk_libraries"],
    package_data={
        "rocm_sdk_libraries": [
            "bin/**/*",
            "lib/**/*",
        ],
    },
    install_requires=[
        "rocm-sdk-core==7.10.0a20251124",  # HIP runtime needed
    ],
)
```

**Step 8: Build the Wheel**

```bash
python -m build --wheel /tmp/staging/rocm_sdk_libraries_gfx94X_dcgpu/

# Creates:
# rocm_sdk_libraries_gfx94X_dcgpu-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl
```

**Step 9: Verify**

```bash
# Check wheel contents
unzip -l rocm_sdk_libraries_gfx94X_dcgpu-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl

Archive:  rocm_sdk_libraries_gfx94X_dcgpu-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2025-11-24 04:30   rocm_sdk_libraries/
      512  2025-11-24 04:30   rocm_sdk_libraries/__init__.py
      235MB 2025-11-24 04:30   rocm_sdk_libraries/lib/librocblas.so.4.0.0
       89MB 2025-11-24 04:30   rocm_sdk_libraries/lib/librocfft.so.1.0.0
      ... (more files)
```

### Building the Other Wheels

The same process repeats for the other packages:

**rocm-sdk-core:** Filters for `core-hip`, `core-runtime`, `base`, includes only `lib` and `run` components

**rocm-sdk-devel:** Includes `dev` components from all artifacts, plus creates symlinks to files in the other packages

**rocm (sdist):** Pure Python package with GPU detection logic and CLI tools

### Upload to S3

After building all wheels, they're uploaded to S3 with an index:

```bash
python ./build_tools/github_actions/upload_python_packages.py \
  --source-dir /tmp/packages/ \
  --s3-bucket therock-nightly-python \
  --s3-prefix v2/gfx94X-dcgpu/ \
  --index-title "ROCm Nightly Python Packages"
```

**S3 Result:**

```
s3://therock-nightly-python/v2/gfx94X-dcgpu/
├── index.html
├── rocm-7.10.0a20251124.tar.gz
├── rocm_sdk_core-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl
├── rocm_sdk_libraries_gfx94X_dcgpu-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl
└── rocm_sdk_devel-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl
```

**The index.html:**

```html
<!DOCTYPE html>
<html>
  <head><title>ROCm Nightly Python Packages</title></head>
  <body>
    <h1>ROCm Nightly Python Packages - gfx94X-dcgpu</h1>
    <a href="rocm-7.10.0a20251124.tar.gz">rocm-7.10.0a20251124.tar.gz</a><br/>
    <a href="rocm_sdk_core-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl">rocm_sdk_core-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl</a><br/>
    <!-- ... more links -->
  </body>
</html>
```

This HTML page is what pip reads when you run:

```bash
pip install rocm[libraries] --index-url https://rocm.nightlies.amd.com/v2/gfx94X-dcgpu/
```

### GPU Detection at Install Time

When you run `pip install rocm[libraries]`, here's what happens:

**Step 1: pip downloads and installs rocm (the sdist)**

```bash
pip install rocm --index-url https://rocm.nightlies.amd.com/v2/gfx94X-dcgpu/
```

**Step 2: rocm's setup.py runs GPU detection**

```python
# From build_tools/packaging/python/templates/rocm/setup.py

# Detect GPU at install time
detected_gpu = determine_target_family()  # Returns "gfx94X-dcgpu"

# Build dependency list based on what user requested
extras_require = {
    "libraries": [
        f"rocm-sdk-libraries-{detected_gpu}=={version}"
    ],
    "devel": [
        f"rocm-sdk-devel=={version}"
    ],
}
```

**GPU detection function:**

```python
# From build_tools/packaging/python/templates/rocm/src/rocm_sdk/_dist_info.py

def discover_current_target_family() -> str | None:
    """Auto-detect GPU using offload-arch tool."""
    try:
        result = subprocess.run(
            ["offload-arch", "--targets"],
            capture_output=True,
            text=True,
            timeout=5
        )
        if result.returncode == 0:
            targets = result.stdout.strip().split("\n")
            if targets:
                # offload-arch returns: gfx942
                # We map to family: gfx94X-dcgpu
                return map_gfx_to_family(targets[0])
    except FileNotFoundError:
        pass
    return None

def determine_target_family() -> str:
    """Determine GPU family with fallback chain."""
    # 1. Check environment variable
    target = os.getenv("ROCM_SDK_TARGET_FAMILY")
    if target:
        return target

    # 2. Auto-detect from hardware
    target = discover_current_target_family()
    if target:
        return target

    # 3. Use default fallback
    return "gfx94X-dcgpu"  # Default
```

**Step 3: pip installs dependencies**

Based on GPU detection, pip now knows to install:

```bash
pip install rocm-sdk-core==7.10.0a20251124
pip install rocm-sdk-libraries-gfx94X-dcgpu==7.10.0a20251124
pip install rocm-sdk-devel==7.10.0a20251124
```

**Result:** User gets GPU-specific libraries automatically, without needing to specify.

---

## Part 5: Stage 2B - Windows Python Packaging

**Workflow:** `.github/workflows/build_windows_python_packages.yml`

This is almost identical to Linux Python packaging, with these differences:

**1. Artifact format:**

```
Input:  therock-core-windows-gfx94X-dcgpu.tar.xz
Output: rocm_sdk_core-7.10.0a20251124-py3-none-win_amd64.whl
```

Notice: `manylinux_2_28_x86_64` → `win_amd64`

**2. Library files:**

Linux uses `.so` files:
```
lib/libamdhip64.so.6.2.0
```

Windows uses `.dll` files:
```
bin/amdhip64.dll
```

**3. No RPATH patching:**

Windows doesn't use RPATH. Instead, DLLs are found via `PATH` environment variable or by being in the same directory.

**4. Executable wrappers:**

Linux: symlinks (`hipcc → hipcc.bin`)

Windows: stub executables (generated C programs that call the real binary)

**Everything else is the same:** same filter logic, same packaging structure, same S3 upload.

---

## Part 6: Stage 2C - Native Linux Packages

### Overview

**Workflow:** `.github/workflows/build_native_linux_packages.yml`

**Input:** Portable artifacts from S3

**Output:** RPM and DEB packages + repository metadata

**Duration:** 20-40 minutes

### What Are Native Linux Packages?

Native packages are what system package managers understand:

- **DEB** = Debian/Ubuntu packages (`.deb` files), installed with `apt`
- **RPM** = Red Hat/Fedora packages (`.rpm` files), installed with `yum` or `dnf`

**Example DEB:**

```
rocm-core_6.2.0-1_amd64.deb
```

**Example RPM:**

```
rocm-core-6.2.0-1.x86_64.rpm
```

### The Packaging Process

**Script:** `build_tools/packaging/linux/build_package.py`

**What it does:**

```bash
# 1. Download artifacts (same as Python packaging)
python ./build_tools/fetch_artifacts.py \
  --run-id 21440027240 \
  --artifact-group gfx94X-dcgpu \
  --output-dir /tmp/artifacts/

# 2. Build DEB packages
python ./build_tools/packaging/linux/build_package.py \
  --artifact-dir /tmp/artifacts/ \
  --package-type deb \
  --rocm-version 6.2.0 \
  --output-dir /tmp/packages/deb/

# 3. Build RPM packages
python ./build_tools/packaging/linux/build_package.py \
  --artifact-dir /tmp/artifacts/ \
  --package-type rpm \
  --rocm-version 6.2.0 \
  --output-dir /tmp/packages/rpm/

# 4. Generate repository metadata
python ./build_tools/packaging/linux/generate_package_indexes.py \
  --package-dir /tmp/packages/deb/ \
  --package-type deb

python ./build_tools/packaging/linux/generate_package_indexes.py \
  --package-dir /tmp/packages/rpm/ \
  --package-type rpm

# 5. Upload to S3
aws s3 sync /tmp/packages/deb/ s3://therock-nightly-packages/gfx94X-dcgpu/deb/
aws s3 sync /tmp/packages/rpm/ s3://therock-nightly-packages/gfx94X-dcgpu/x86_64/
```

### Deep Dive: Building DEB Packages

Let's trace how a DEB package is created for rocBLAS.

**Step 1: Extract Artifact**

```bash
cd /tmp/staging/rocm-blas/

tar -xJf /tmp/artifacts/therock-blas-linux-gfx94X-dcgpu.tar.xz
```

**Step 2: Split into Package Components**

The artifact contains multiple components (lib, dev, test). We create separate packages:

```
rocm-blas            # Runtime libraries (lib component)
rocm-blas-dev        # Headers and CMake files (dev component)
rocm-blas-tests      # Benchmarks and tests (test component)
```

**Step 3: Create DEB Directory Structure**

For `rocm-blas` (runtime):

```
/tmp/staging/rocm-blas_6.2.0-1_amd64/
├── DEBIAN/
│   ├── control          # Package metadata
│   ├── postinst         # Post-installation script
│   └── prerm            # Pre-removal script
├── opt/rocm-6.2.0/
│   ├── bin/
│   │   └── rocblas/
│   │       └── library/
│   │           └── TensileLibrary_gfx94X.dat
│   └── lib/
│       ├── librocblas.so.4.0.0
│       ├── librocblas.so.4 → librocblas.so.4.0.0
│       └── librocblas.so → librocblas.so.4
```

**DEBIAN/control:**

```
Package: rocm-blas
Version: 6.2.0-1
Architecture: amd64
Maintainer: ROCm Development <rocm-dev@amd.com>
Description: ROCm BLAS library
 Provides Basic Linear Algebra Subprograms (BLAS) for AMD GPUs
Depends: rocm-core (>= 6.2.0)
Section: devel
Priority: optional
```

**DEBIAN/postinst:**

```bash
#!/bin/bash
set -e

# Update dynamic linker cache so libraries are found
ldconfig /opt/rocm-6.2.0/lib

# Create symlink for versioned ROCm
if [ ! -e /opt/rocm ]; then
    ln -s /opt/rocm-6.2.0 /opt/rocm
fi

exit 0
```

**Step 4: Build the DEB**

```bash
dpkg-deb --build /tmp/staging/rocm-blas_6.2.0-1_amd64/

# Creates:
# /tmp/packages/deb/rocm-blas_6.2.0-1_amd64.deb
```

**Step 5: Repeat for Other Components**

Build `rocm-blas-dev`:

```
/tmp/staging/rocm-blas-dev_6.2.0-1_amd64/
├── DEBIAN/
│   └── control
└── opt/rocm-6.2.0/
    ├── include/
    │   └── rocblas/
    │       └── rocblas.h
    └── lib/cmake/
        └── rocblas/
            └── rocblas-config.cmake
```

**DEBIAN/control:**

```
Package: rocm-blas-dev
Version: 6.2.0-1
Architecture: amd64
Description: ROCm BLAS development files
 Headers and CMake files for developing with rocBLAS
Depends: rocm-blas (= 6.2.0-1), rocm-core-dev (>= 6.2.0)
```

**Step 6: Create Repository Metadata**

DEB repositories need a `Packages.gz` file that lists all available packages:

```bash
cd /tmp/packages/deb/

dpkg-scanpackages . /dev/null | gzip -9c > Packages.gz
```

**Packages.gz contents:**

```
Package: rocm-blas
Version: 6.2.0-1
Architecture: amd64
Filename: ./rocm-blas_6.2.0-1_amd64.deb
Size: 235840000
SHA256: a1b2c3d4e5f6...
Description: ROCm BLAS library

Package: rocm-blas-dev
Version: 6.2.0-1
Architecture: amd64
Filename: ./rocm-blas-dev_6.2.0-1_amd64.deb
Size: 450000
SHA256: f6e5d4c3b2a1...
Description: ROCm BLAS development files
```

**Step 7: Upload to S3**

```
s3://therock-nightly-packages/gfx94X-dcgpu/deb/
├── Packages.gz
├── rocm-blas_6.2.0-1_amd64.deb
├── rocm-blas-dev_6.2.0-1_amd64.deb
├── rocm-core_6.2.0-1_amd64.deb
├── rocm-core-dev_6.2.0-1_amd64.deb
└── ... (all other packages)
```

### Deep Dive: Building RPM Packages

The RPM process is similar but uses different tools:

**RPM Directory Structure:**

```
/tmp/staging/rpm/
├── BUILD/              # Build workspace
├── RPMS/              # Output directory
├── SOURCES/           # Source tarballs
├── SPECS/             # .spec files (recipes)
└── SRPMS/             # Source RPMs
```

**rocm-blas.spec:**

```spec
Name:           rocm-blas
Version:        6.2.0
Release:        1%{?dist}
Summary:        ROCm BLAS library
License:        MIT
URL:            https://github.com/ROCm/rocBLAS

Requires:       rocm-core >= 6.2.0

%description
Provides Basic Linear Algebra Subprograms (BLAS) for AMD GPUs

%install
# Copy files from artifact
mkdir -p %{buildroot}/opt/rocm-%{version}/lib
cp -r %{_builddir}/lib/* %{buildroot}/opt/rocm-%{version}/lib/

%files
/opt/rocm-%{version}/lib/librocblas.so*
/opt/rocm-%{version}/bin/rocblas/library/*

%post
/sbin/ldconfig /opt/rocm-%{version}/lib

%postun
/sbin/ldconfig
```

**Build the RPM:**

```bash
rpmbuild -bb rocm-blas.spec

# Creates:
# /tmp/staging/rpm/RPMS/x86_64/rocm-blas-6.2.0-1.x86_64.rpm
```

**Generate repository metadata:**

```bash
cd /tmp/packages/rpm/

createrepo_c .

# Creates:
# repodata/
# ├── repomd.xml
# ├── primary.xml.gz
# ├── filelists.xml.gz
# └── other.xml.gz
```

**Upload to S3:**

```
s3://therock-nightly-packages/gfx94X-dcgpu/x86_64/
├── repodata/
│   ├── repomd.xml
│   └── primary.xml.gz
├── rocm-blas-6.2.0-1.x86_64.rpm
├── rocm-blas-dev-6.2.0-1.x86_64.rpm
└── ... (all other RPMs)
```

### How Users Install Native Packages

**For DEB (Ubuntu/Debian):**

```bash
# Add ROCm repository
echo "deb [trusted=yes] https://rocm.nightlies.amd.com/packages/gfx94X-dcgpu/deb/ ./" \
  | sudo tee /etc/apt/sources.list.d/rocm.list

# Update package list
sudo apt update

# Install ROCm
sudo apt install rocm-blas rocm-blas-dev
```

**For RPM (RHEL/Fedora):**

```bash
# Add ROCm repository
sudo tee /etc/yum.repos.d/rocm.repo <<EOF
[rocm]
name=ROCm Nightly
baseurl=https://rocm.nightlies.amd.com/packages/gfx94X-dcgpu/x86_64/
enabled=1
gpgcheck=0
EOF

# Install ROCm
sudo yum install rocm-blas rocm-blas-dev
```

---

## Part 7: Distribution via S3 and CloudFront

### S3 Bucket Organization

TheRock uses different S3 buckets for different release types:

| Release Type | Use Case | S3 Buckets | Retention |
|---|---|---|---|
| **CI** | Temporary test builds | `therock-ci-artifacts` | 7 days |
| **Nightly** | Automated daily builds | `therock-nightly-{tarball,python,packages}` | 30 days |
| **Dev** | Development snapshots | `therock-dev-{tarball,python,packages}` | 90 days |
| **Prerelease** | Release candidates | `therock-prerelease-{tarball,python,packages}` | Permanent |
| **Release** | Official releases | `therock-release-{tarball,python,packages}` | Permanent |

### CloudFront CDN

S3 buckets are fronted by CloudFront (AWS's CDN) for fast global access:

| S3 Bucket | CloudFront URL |
|---|---|
| `therock-nightly-python` | `https://rocm.nightlies.amd.com/` |
| `therock-dev-python` | `https://rocm.devreleases.amd.com/` |
| `therock-prerelease-python` | `https://rocm.prereleases.amd.com/` |

**Why CloudFront?**

- **Faster downloads:** Content cached at edge locations worldwide
- **HTTPS:** Secure package downloads
- **Cheaper:** S3 egress costs reduced

### URL Structure

**Python packages:**

```
# Nightly
https://rocm.nightlies.amd.com/v2/gfx94X-dcgpu/index.html

# Dev
https://rocm.devreleases.amd.com/v2/gfx94X-dcgpu/index.html

# Prerelease
https://rocm.prereleases.amd.com/whl/index.html
```

**Tarballs:**

```
# Nightly
https://rocm.nightlies.amd.com/tarball/therock-dist-linux-gfx94X-dcgpu-7.10.0a20251124.tar.gz

# Dev
https://rocm.devreleases.amd.com/tarball/therock-dist-linux-gfx94X-dcgpu-7.10.0.dev0+f689a8ea.tar.gz
```

**Native packages:**

```
# Nightly DEB
https://rocm.nightlies.amd.com/packages/gfx94X-dcgpu/deb/Packages.gz

# Nightly RPM
https://rocm.nightlies.amd.com/packages/gfx94X-dcgpu/x86_64/repodata/repomd.xml
```

---

## Part 8: Release vs Nightly vs Dev Builds

### Build Types Comparison

| Aspect | CI | Nightly | Dev | Prerelease | Release |
|---|---|---|---|---|---|
| **Trigger** | Manual/PR | Daily 4AM UTC | Manual | Manual | Manual |
| **Version** | ADHOCBUILD | `7.10.0a20251124` | `7.10.0.dev0+abc123` | `7.10.0rc2` | `7.10.0` |
| **S3 Bucket** | therock-ci-* | therock-nightly-* | therock-dev-* | therock-prerelease-* | therock-release-* |
| **Retention** | 7 days | 30 days | 90 days | Permanent | Permanent |
| **Downstream** | No | Yes | No | Yes | Yes |
| **Purpose** | Testing | Users | Developers | Beta testers | Production |

**"Downstream"** means: triggers PyTorch, JAX, and native package builds

### Version String Formats

**Computed by:** `build_tools/compute_rocm_package_version.py`

**Nightly:**

```
7.10.0a20251124
```

- `7.10.0` = base version from `VERSION.txt`
- `a` = alpha (pre-release marker)
- `20251124` = date (YYYYMMDD)

**Dev:**

```
7.10.0.dev0+f689a8ea
```

- `7.10.0` = base version
- `.dev0` = development marker
- `+f689a8ea` = git commit hash (first 8 chars)

**Prerelease:**

```
7.10.0rc2
```

- `7.10.0` = base version
- `rc2` = release candidate 2

**Release:**

```
7.10.0
```

- Just the version, no suffix

### Workflow Comparison

**Nightly Workflow:** `release_portable_linux_packages.yml`

```yaml
on:
  schedule:
    - cron: "0 4 * * *"  # 4 AM UTC daily

jobs:
  setup_metadata:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - name: Compute version
        id: version
        run: |
          VERSION=$(python ./build_tools/compute_rocm_package_version.py --release-type nightly)
          echo "version=$VERSION" >> $GITHUB_OUTPUT

  build_artifacts:
    needs: setup_metadata
    strategy:
      matrix:
        gpu_family: [gfx94X-dcgpu, gfx950-dcgpu, gfx101X-dgpu]
    uses: ./.github/workflows/build_portable_linux_artifacts.yml
    with:
      amdgpu_families: ${{ matrix.gpu_family }}
      package_version: ${{ needs.setup_metadata.outputs.version }}

  build_python_packages:
    needs: [setup_metadata, build_artifacts]
    strategy:
      matrix:
        gpu_family: [gfx94X-dcgpu, gfx950-dcgpu, gfx101X-dgpu]
    uses: ./.github/workflows/build_portable_linux_python_packages.yml
    with:
      artifact_run_id: ${{ github.run_id }}
      artifact_group: ${{ matrix.gpu_family }}
      package_version: ${{ needs.setup_metadata.outputs.version }}
      s3_bucket: therock-nightly-python
      s3_prefix: v2/${{ matrix.gpu_family }}/

  build_pytorch_wheels:
    needs: [setup_metadata, build_python_packages]
    uses: ./.github/workflows/release_portable_linux_pytorch_wheels.yml
    # ... (PyTorch build config)

  build_jax_wheels:
    needs: [setup_metadata, build_python_packages]
    uses: ./.github/workflows/release_portable_linux_jax_wheels.yml
    # ... (JAX build config)

  build_native_packages:
    needs: [setup_metadata, build_artifacts]
    uses: ./.github/workflows/build_native_linux_packages.yml
    # ... (native package config)
```

**Dev Workflow:** Similar, but with different S3 bucket and version format

**CI Nightly Workflow:** `ci_nightly.yml`

```yaml
on:
  schedule:
    - cron: "0 2 * * *"  # 2 AM UTC daily

jobs:
  build_and_test:
    strategy:
      matrix:
        gpu_family: [gfx94X-dcgpu]
    uses: ./.github/workflows/build_portable_linux_artifacts.yml
    with:
      amdgpu_families: ${{ matrix.gpu_family }}
      # Artifacts stay in therock-ci-artifacts (temporary)
      # NO downstream builds (PyTorch, JAX, native packages)
```

**Key difference:** CI nightly is for testing code health. Release nightly is for users.

---

## Part 9: Downstream Framework Builds

Once portable artifacts are built, several downstream workflows can trigger.

### PyTorch Wheels

**Workflow:** `release_portable_linux_pytorch_wheels.yml`

**What it does:**

1. Downloads ROCm Python packages from S3
2. Builds PyTorch from source against ROCm
3. Creates PyTorch wheels for multiple Python versions

**Matrix expansion:**

```yaml
strategy:
  matrix:
    python_version: ["3.10", "3.11", "3.12", "3.13"]
    pytorch_version: ["2.8", "2.9", "2.10", "nightly"]
    gpu_family: [gfx94X-dcgpu, gfx950-dcgpu]
```

**Total jobs:** 4 Python × 4 PyTorch × 2 GPU = 32 PyTorch wheels

**Build process:**

```bash
# 1. Install ROCm Python packages
pip install rocm[libraries] \
  --index-url https://rocm.nightlies.amd.com/v2/gfx94X-dcgpu/

# 2. Set environment for PyTorch build
export ROCM_HOME=$(rocm-sdk path --root)
export CMAKE_PREFIX_PATH=$(rocm-sdk path --cmake)

# 3. Build PyTorch
git clone https://github.com/pytorch/pytorch.git
cd pytorch
python setup.py bdist_wheel

# 4. Upload wheel
# torch-2.8.0+rocm7.10.0-cp311-cp311-manylinux_2_28_x86_64.whl
```

**Output location:**

```
s3://therock-nightly-python/v2/gfx94X-dcgpu/
└── torch-2.8.0+rocm7.10.0-cp311-cp311-manylinux_2_28_x86_64.whl
```

Users install with:

```bash
pip install torch --index-url https://rocm.nightlies.amd.com/v2/gfx94X-dcgpu/
```

### JAX Wheels

**Workflow:** `release_portable_linux_jax_wheels.yml`

Similar to PyTorch, but builds JAX:

**Matrix:**

```yaml
strategy:
  matrix:
    python_version: ["3.10", "3.11", "3.12"]
    jax_version: ["0.4.28", "0.4.29", "latest"]
```

**Output:**

```
jax-0.4.28+rocm7.10.0-cp311-cp311-manylinux_2_28_x86_64.whl
```

---

## Part 10: Complete Example - rocBLAS Journey

Let's trace a single library (rocBLAS) through the entire packaging pipeline.

### Stage 0: Source Code

```
BUILD_TOPOLOGY.toml:

[source_sets.math-libs.rocBLAS]
url = "https://github.com/ROCm/rocBLAS.git"
branch = "develop"

[artifacts.blas]
artifact_group = "math-libs"
type = "target-specific"
artifact_deps = ["core-hip"]
```

### Stage 1: Building

```bash
# CI runs:
cmake -B build -GNinja -DTHEROCK_AMDGPU_FAMILIES=gfx94X-dcgpu
ninja -C build rocBLAS

# Build output:
build/math-libs/BLAS/rocBLAS/stage/
├── bin/
│   ├── rocblas-bench
│   ├── rocblas-test
│   └── rocblas/library/TensileLibrary_gfx94X.dat
├── lib/
│   ├── librocblas.so.4.0.0        (235 MB)
│   ├── librocblas.so.4 → librocblas.so.4.0.0
│   └── librocblas.so → librocblas.so.4
├── include/
│   └── rocblas/rocblas.h
└── lib/cmake/
    └── rocblas/rocblas-config.cmake

# Artifact creation:
ninja -C build artifacts

# Creates:
build/artifacts/therock-blas-linux-gfx94X-dcgpu.tar.xz  (65 MB compressed)

# Upload:
s3://therock-ci-artifacts/21440027240-linux/therock-blas-linux-gfx94X-dcgpu.tar.xz
```

### Stage 2A: Python Packaging

```bash
# Download artifact
aws s3 cp s3://therock-ci-artifacts/21440027240-linux/therock-blas-linux-gfx94X-dcgpu.tar.xz .

# Extract
tar -xJf therock-blas-linux-gfx94X-dcgpu.tar.xz

# Filter (keep lib component only for runtime wheel)
# Remove symlinks (keep SONAME only)
# Patch RPATH
# Add Python wrapper

# Build wheel
python -m build --wheel

# Creates:
rocm_sdk_libraries_gfx94X_dcgpu-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl

# Inside wheel:
rocm_sdk_libraries/
├── __init__.py
└── lib/
    └── librocblas.so.4.0.0  (235 MB)

# Upload:
s3://therock-nightly-python/v2/gfx94X-dcgpu/rocm_sdk_libraries_gfx94X_dcgpu-7.10.0a20251124-py3-none-manylinux_2_28_x86_64.whl
```

### Stage 2C: Native Packaging

**DEB:**

```bash
# Download same artifact
aws s3 cp s3://therock-ci-artifacts/21440027240-linux/therock-blas-linux-gfx94X-dcgpu.tar.xz .

# Extract
tar -xJf therock-blas-linux-gfx94X-dcgpu.tar.xz

# Split into components:
# - rocm-blas (lib component)
# - rocm-blas-dev (dev component)
# - rocm-blas-tests (test component)

# Build rocm-blas DEB:
dpkg-deb --build rocm-blas_6.2.0-1_amd64/

# Creates:
rocm-blas_6.2.0-1_amd64.deb

# Inside DEB:
/opt/rocm-6.2.0/
├── bin/rocblas/library/TensileLibrary_gfx94X.dat
└── lib/
    ├── librocblas.so.4.0.0  (235 MB)
    ├── librocblas.so.4 → librocblas.so.4.0.0
    └── librocblas.so → librocblas.so.4

# Upload:
s3://therock-nightly-packages/gfx94X-dcgpu/deb/rocm-blas_6.2.0-1_amd64.deb
```

**RPM:**

```bash
# Same artifact, different packaging

rpmbuild -bb rocm-blas.spec

# Creates:
rocm-blas-6.2.0-1.x86_64.rpm

# Upload:
s3://therock-nightly-packages/gfx94X-dcgpu/x86_64/rocm-blas-6.2.0-1.x86_64.rpm
```

### End User Installation

**Python:**

```bash
pip install rocm[libraries] --index-url https://rocm.nightlies.amd.com/v2/gfx94X-dcgpu/

# Result:
~/.local/lib/python3.11/site-packages/rocm_sdk_libraries/lib/librocblas.so.4.0.0
```

**DEB:**

```bash
sudo apt install rocm-blas

# Result:
/opt/rocm-6.2.0/lib/librocblas.so.4.0.0
```

**RPM:**

```bash
sudo yum install rocm-blas

# Result:
/opt/rocm-6.2.0/lib/librocblas.so.4.0.0
```

**All three installations are functionally identical!** They install the same 235 MB library file, just in different locations with different package managers.

---

## Part 11: Storage Efficiency - No Duplication

### The Problem

We produce multiple packages from the same build. Are we storing files multiple times?

**Example:** `librocblas.so.4.0.0` (235 MB)

This file appears in:
- `rocm-sdk-libraries` wheel
- `rocm-sdk-devel` wheel
- `rocm-blas` DEB package
- `rocm-blas` RPM package

Are we storing 4 × 235 MB = 940 MB?

### The Solution

**For Python wheels:** symlinks

The `rocm-sdk-devel` package doesn't duplicate files. Instead, it creates symlinks to the `rocm-sdk-libraries` package:

```python
# In build_tools/_therock_utils/py_packaging.py

def _populate_devel_file(self, relpath: str, dest_path: Path):
    """Add file to devel package."""

    # Check if this file is already in a runtime package
    if self.params.files.has(relpath):
        # Get which package has it
        populated_package, populated_path = \
            self.params.files.materialized_relpaths[relpath]

        # Create relative symlink to that package
        relpath_segment_count = len(Path(relpath).parts)
        backtrack_path = Path(*([".."] * relpath_segment_count))
        link_target = backtrack_path / populated_package.platform_dir.name / relpath

        dest_path.symlink_to(link_target)
        return

    # File not in runtime package, copy it to devel
    shutil.copy2(src_path, dest_path)
```

**Result:**

```
rocm_sdk_libraries/lib/librocblas.so.4.0.0   # 235 MB actual file
rocm_sdk_devel/lib/librocblas.so.4.0.0       # 0 bytes (symlink)
```

**Total storage:** 235 MB, not 470 MB!

**For native packages:** no duplication

RPM and DEB packages don't share files with each other. Each is independent. But users only install ONE (either RPM or DEB, not both), so no duplication on user systems.

**For CI artifacts:** compressed

The `.tar.xz` files are compressed:

```
build/math-libs/BLAS/rocBLAS/stage/  # 890 MB uncompressed
therock-blas-linux-gfx94X.tar.xz     #  65 MB compressed (7.3% ratio)
```

### Storage Costs

**CI artifacts (7 days):**

```
One build for gfx94X-dcgpu:
- Linux artifacts: ~15 GB compressed
- Windows artifacts: ~12 GB compressed
Total: ~27 GB × 7 days = ~189 GB
```

**Nightly releases (30 days):**

```
One nightly build:
- Tarballs: ~8 GB (compressed dist)
- Python wheels: ~6 GB (includes all GPU families)
- Native packages: ~15 GB (DEB + RPM)
Total: ~29 GB × 30 days = ~870 GB
```

**Actual S3 costs:** ~$20-30/month for nightly builds (S3 Standard tier)

---

## Part 12: Summary and Quick Reference

### The Complete Pipeline

```
Source Code
    ↓ (4-8 hours)
Portable Artifacts (.tar.xz)
    ↓ (parallel, 10-40 min each)
    ├─→ Python Wheels (.whl)
    ├─→ Native DEB (.deb)
    └─→ Native RPM (.rpm)
    ↓
S3 Storage
    ↓
CloudFront CDN
    ↓
End Users
```

### Key Files Reference

| File | Purpose |
|---|---|
| `BUILD_TOPOLOGY.toml` | Defines all components and dependencies |
| `math-libs/BLAS/artifact-blas.toml` | Defines how to slice rocBLAS into components |
| `.github/workflows/build_portable_linux_artifacts.yml` | Builds portable Linux artifacts |
| `.github/workflows/build_portable_linux_python_packages.yml` | Creates Python wheels |
| `.github/workflows/build_native_linux_packages.yml` | Creates DEB/RPM packages |
| `build_tools/build_python_packages.py` | Main Python packaging script |
| `build_tools/packaging/linux/build_package.py` | Main native packaging script |
| `build_tools/compute_rocm_package_version.py` | Computes version strings |

### Package Manager Commands

**Python (pip):**

```bash
# Install from nightly
pip install rocm[libraries,devel] \
  --index-url https://rocm.nightlies.amd.com/v2/gfx94X-dcgpu/

# Install from dev
pip install rocm[libraries,devel] \
  --index-url https://rocm.devreleases.amd.com/v2/gfx94X-dcgpu/
```

**DEB (apt):**

```bash
# Add repository
echo "deb [trusted=yes] https://rocm.nightlies.amd.com/packages/gfx94X-dcgpu/deb/ ./" \
  | sudo tee /etc/apt/sources.list.d/rocm.list

# Install
sudo apt update && sudo apt install rocm
```

**RPM (yum/dnf):**

```bash
# Add repository
sudo tee /etc/yum.repos.d/rocm.repo <<EOF
[rocm]
name=ROCm Nightly
baseurl=https://rocm.nightlies.amd.com/packages/gfx94X-dcgpu/x86_64/
enabled=1
gpgcheck=0
EOF

# Install
sudo yum install rocm
```

### S3 Bucket Quick Reference

| Purpose | Tarball | Python | Native Packages |
|---|---|---|---|
| **CI** | therock-ci-artifacts | (same) | N/A |
| **Nightly** | therock-nightly-tarball | therock-nightly-python | therock-nightly-packages |
| **Dev** | therock-dev-tarball | therock-dev-python | therock-dev-packages |
| **Prerelease** | therock-prerelease-tarball | therock-prerelease-python | therock-prerelease-packages |

### Version Format Quick Reference

| Type | Example | Pattern |
|---|---|---|
| **Nightly** | `7.10.0a20251124` | `{version}a{YYYYMMDD}` |
| **Dev** | `7.10.0.dev0+f689a8ea` | `{version}.dev0+{commit}` |
| **Prerelease** | `7.10.0rc2` | `{version}rc{N}` |
| **Release** | `7.10.0` | `{version}` |

### GPU Family Examples

| Family | Description | Example GPUs |
|---|---|---|
| `gfx94X-dcgpu` | MI300 series datacenter | MI300A, MI300X |
| `gfx950-dcgpu` | MI340 series datacenter | MI340 |
| `gfx101X-dgpu` | Consumer RDNA2/3 | RX 6000/7000 |
| `gfx1151` | APU RDNA3 | Ryzen 7000 series |
| `gfx120X-all` | RDNA4 | RX 8000 series |

---

## Conclusion

This document covered the complete TheRock packaging pipeline:

1. **BUILD_TOPOLOGY.toml** organizes source code and defines artifacts
2. **Stage 1** builds portable artifacts (`.tar.xz` files) from source
3. **Stage 2** transforms artifacts into three package types:
   - **Python wheels** for `pip install`
   - **DEB packages** for `apt install`
   - **RPM packages** for `yum install`
4. **S3 + CloudFront** distribute packages to users worldwide
5. **Different release types** (CI, nightly, dev, prerelease) use the same pipeline with different configurations

**Key insight:** Build once (slow), package many ways (fast), deliver everywhere.

All package types are built from the same source and provide the same ROCm functionality. The only difference is the delivery mechanism: Python package managers, native Linux package managers, or direct tarballs.
