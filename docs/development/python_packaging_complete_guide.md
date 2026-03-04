# TheRock Python Packaging: Complete Self-Contained Guide

**Last Updated:** March 2026

This document is a complete, self-contained guide to how ROCm is packaged as Python wheels. You don't need to read any other documentation - everything you need is here.

---

## What You'll Learn

1. How ROCm source code becomes Python packages you can `pip install`
2. What BUILD_TOPOLOGY.toml is and why it matters
3. How component artifacts are created and organized
4. How Python packages are built from artifacts
5. How pip installs the right packages for your GPU
6. Why files aren't duplicated between packages

---

## Table of Contents

- [Part 1: The Big Picture](#part-1-big-picture)
- [Part 2: BUILD_TOPOLOGY.toml - The Master Blueprint](#part-2-topology)
- [Part 3: Building Component Artifacts](#part-3-artifacts)
- [Part 4: Creating Python Packages](#part-4-python-packages)
- [Part 5: User Installation with pip](#part-5-pip-install)
- [Part 6: Storage Efficiency and Symlinks](#part-6-storage)
- [Appendix: Complete rocBLAS Example](#appendix-complete-example)

---

<a name="part-1-big-picture"></a>
## Part 1: The Big Picture

### What is TheRock?

TheRock is a CMake "super-project" that builds the entire ROCm software stack from source. ROCm (Radeon Open Compute) is AMD's GPU computing platform - similar to NVIDIA's CUDA.

### Why Multiple Packages?

ROCm is huge (~10GB+ built). Not all users need everything:
- Some just need HIP runtime (to run PyTorch)
- Some need math libraries (rocBLAS, rocFFT for ML)
- Some need development files (headers, CMake configs to build code)

So we split it into **4 Python packages**:

1. **`rocm`** - Selector package (detects your GPU)
2. **`rocm-sdk-core`** - HIP runtime (required, ~500MB)
3. **`rocm-sdk-libraries-{gpu}`** - Math libraries (optional, ~2GB, GPU-specific)
4. **`rocm-sdk-devel`** - Headers and build tools (optional, ~300MB)

### The Journey: Source Code → pip install

```
┌───────────────────────────────────────────────────────────────┐
│ Step 0: BUILD_TOPOLOGY.toml                                   │
│ Master configuration defining what gets built                 │
└─────────────────────────┬─────────────────────────────────────┘
                          │
                          ↓
┌───────────────────────────────────────────────────────────────┐
│ Step 1: Build Source Code (4-8 hours)                        │
│ Input:  C++ source files                                     │
│ Output: Component artifacts (*.tar.xz files)                 │
│ Where:  S3 bucket (s3://therock-ci-artifacts/)               │
└─────────────────────────┬─────────────────────────────────────┘
                          │
                          ↓
┌───────────────────────────────────────────────────────────────┐
│ Step 2: Package as Python Wheels (10-20 minutes)             │
│ Input:  Component artifacts from Step 1                      │
│ Output: Python wheels (*.whl files)                          │
│ Where:  S3 bucket (s3://therock-nightly-python/)             │
└─────────────────────────┬─────────────────────────────────────┘
                          │
                          ↓
┌───────────────────────────────────────────────────────────────┐
│ Step 3: User Installation (2-5 minutes)                      │
│ Input:  pip install rocm[libraries]                          │
│ Output: ROCm installed in Python virtual environment         │
│ Where:  User's .venv/lib/python3.*/site-packages/            │
└───────────────────────────────────────────────────────────────┘
```

**Key insight:** Steps 1 and 2 happen in CI (continuous integration). Step 3 is when users install via pip.

---

<a name="part-2-topology"></a>
## Part 2: BUILD_TOPOLOGY.toml - The Master Blueprint

### What is BUILD_TOPOLOGY.toml?

Think of it as the **table of contents for the entire ROCm build**. It's a single file that defines:
- What gets built (artifacts)
- How they're organized (groups)
- What depends on what (dependencies)
- Which source code is needed (git submodules)

**Location:** Root of repository at `/BUILD_TOPOLOGY.toml`

### Why Does It Exist?

Without BUILD_TOPOLOGY.toml, the information would be scattered across hundreds of CMakeLists.txt files. With it, everything is in one place.

### The Hierarchy

BUILD_TOPOLOGY.toml defines 4 levels:

```
Level 1: SOURCE SETS
         ↓ (defines which git repos to clone)
Level 2: BUILD STAGES
         ↓ (defines CI job groups)
Level 3: ARTIFACT GROUPS
         ↓ (defines logical collections)
Level 4: ARTIFACTS
         ↓ (defines individual build outputs)
```

Let me show you a real example for rocBLAS (a matrix math library).

### Real Example: rocBLAS in BUILD_TOPOLOGY.toml

#### Level 1: Source Set (Which Code Do We Need?)

```toml
[source_sets.rocm-libraries]
description = "ROCm libraries monorepo (math libs)"
submodules = ["rocm-libraries"]
```

**Translation:** To build math libraries, we need to clone the `rocm-libraries` git submodule.

**Physical location on disk:**
```
TheRock/
└── math-libs/
    └── rocm-libraries/    ← This is a git submodule
        ├── hipBLAS/       ← Contains rocBLAS wrapper
        ├── rocBLAS/       ← Contains rocBLAS source code
        │   ├── library/
        │   │   └── src/
        │   │       └── blas1/
        │   │           └── rocblas_axpy.cpp  ← Actual C++ code
        │   └── CMakeLists.txt
        ├── rocFFT/
        └── rocRAND/
```

#### Level 2: Build Stage (When/How to Build?)

```toml
[build_stages.math-libs]
description = "Math libraries (BLAS, FFT, RAND, SOLVER, SPARSE)"
artifact_groups = ["math-libs"]
type = "per-arch"
```

**Translation:**
- Create a CI job called "math-libs"
- Build the "math-libs" artifact group in this job
- `type = "per-arch"` means build it separately for each GPU (gfx1150, gfx94x, gfx950)

**What CI does:** Creates 3 parallel jobs:
```yaml
job: math-libs-gfx1150    # For Radeon RX 7000 series
job: math-libs-gfx94x     # For MI300 data center GPUs
job: math-libs-gfx950     # For MI350 data center GPUs
```

#### Level 3: Artifact Group (What's Included?)

```toml
[artifact_groups.math-libs]
description = "Math libraries (BLAS, FFT, RAND, etc.)"
type = "per-arch"
artifact_group_deps = ["hip-runtime", "profiler-core"]
source_sets = ["rocm-libraries", "rocm-systems", "math-libs"]
```

**Translation:**
- This group contains multiple artifacts: blas, fft, rand, etc.
- **Dependencies:** Must build `hip-runtime` and `profiler-core` first
- **Source code needed:** Clone `rocm-libraries`, `rocm-systems`, `math-libs` submodules

#### Level 4: Artifact (Individual Component)

```toml
[artifacts.blas]
artifact_group = "math-libs"
type = "target-specific"
artifact_deps = ["core-hip", "rocprofiler-sdk"]
feature_group = "MATH_LIBS"
```

**Translation:**
- Artifact name: `blas`
- Part of the `math-libs` group
- **Type:** `target-specific` = built once per GPU architecture, kept separate
- **Dependencies:** Needs `core-hip` and `rocprofiler-sdk` artifacts built first
- **CMake control:** Users can disable with `-DTHEROCK_ENABLE_MATH_LIBS=OFF`

### How BUILD_TOPOLOGY.toml is Used

#### Used by CMake (at build time)

When you run:
```bash
cmake -B build -GNinja
```

CMake does this:
1. Runs Python script: `build_tools/topology_to_cmake.py`
2. Reads `BUILD_TOPOLOGY.toml`
3. Generates `build/cmake/topology_generated.cmake`

**What gets generated:**

```cmake
# Lists all artifacts
set(THEROCK_TOPOLOGY_ARTIFACTS
  "sysdeps" "base" "amd-llvm" "core-hip" "blas" "fft" ...
)

# Defines artifact properties
set(THEROCK_ARTIFACT_TYPE_blas "target-specific")
set(THEROCK_ARTIFACT_GROUP_blas "math-libs")
set(THEROCK_ARTIFACT_DEPS_blas "core-hip;rocprofiler-sdk")

# Creates CMake targets
add_custom_target(artifact-blas COMMENT "Building artifact blas")
add_custom_target(artifact-group-math-libs)
add_dependencies(artifact-group-math-libs artifact-blas artifact-fft ...)

# Creates user-controllable feature flags
# User can run: cmake -B build -DTHEROCK_ENABLE_MATH_LIBS=OFF
therock_add_feature(MATH_LIBS
  "Math libraries (BLAS, FFT, RAND)"
  ON  # Enabled by default
)
```

#### Used by CI (at pipeline time)

CI script `configure_ci.py` reads the topology:

```python
from _therock_utils.build_topology import BuildTopology

topology = BuildTopology.from_toml("BUILD_TOPOLOGY.toml")

# Get all build stages
for stage in topology.get_build_stages():
    if stage.type == "per-arch":
        # Create separate job for each GPU
        for gpu in ["gfx1150", "gfx94x", "gfx950"]:
            create_ci_job(name=f"{stage.name}-{gpu}")
```

**Generated CI matrix:**
```json
{
  "include": [
    {"stage": "math-libs", "gpu": "gfx1150"},
    {"stage": "math-libs", "gpu": "gfx94x"},
    {"stage": "math-libs", "gpu": "gfx950"}
  ]
}
```

#### Used by Python Packaging (at package time)

Python packaging script uses artifact names from topology:

```python
# build_tools/build_python_packages.py
def libraries_artifact_filter(target_family: str, an: ArtifactName) -> bool:
    return (
        an.name in ["blas", "fft", "rand", ...]  # ← Names from topology
        and an.component in ["lib"]
    )
```

**Summary:** BUILD_TOPOLOGY.toml is the single source of truth that drives CMake, CI, and Python packaging.

---

<a name="part-3-artifacts"></a>
## Part 3: Building Component Artifacts

### What Are Component Artifacts?

A **component artifact** is a compressed archive (`.tar.xz` file) containing a specific piece of ROCm. Examples:
- `core-hip_lib_generic.tar.xz` - HIP runtime libraries
- `blas_lib_gfx1150.tar.xz` - rocBLAS libraries for gfx1150 GPUs
- `blas_dev_gfx1150.tar.xz` - rocBLAS headers and CMake files

### Why Component Artifacts?

Component artifacts are the **reusable intermediate format**. Once built, they can be:
- Downloaded by later CI stages (for dependent builds)
- Packaged into Python wheels
- Packaged into RPM/DEB packages
- Used for PyTorch/JAX wheel builds

**Key point:** Build once, reuse many times.

### The Build Process

Let me show you the complete process for rocBLAS.

#### Step 1: Source Code

**File on disk:**
```
math-libs/rocm-libraries/rocBLAS/library/src/blas1/rocblas_axpy.cpp
```

**Example code (simplified):**
```cpp
// rocblas_axpy.cpp - Computes y = alpha*x + y
rocblas_status rocblas_saxpy(
    rocblas_handle handle,
    rocblas_int n,
    const float* alpha,
    const float* x,
    rocblas_int incx,
    float* y,
    rocblas_int incy
) {
    // Implementation that launches GPU kernels
    return hipLaunchKernelGGL(saxpy_kernel, ...);
}
```

#### Step 2: CMake Build

**Command:**
```bash
cmake -B build -GNinja -DTHEROCK_AMDGPU_FAMILIES=gfx1150
ninja -C build
```

**What happens:**
- CMake reads `BUILD_TOPOLOGY.toml` (via topology_to_cmake.py)
- Knows to build the `blas` artifact
- Compiles C++ source → binaries
- Installs to "stage directory"

**Stage directory structure:**
```
build/math-libs/BLAS/rocBLAS/stage/
├── bin/
│   ├── rocblas-bench                      ← Benchmark executable
│   ├── rocblas-test                       ← Test executable
│   └── rocblas/
│       └── library/
│           └── TensileLibrary_gfx1150.dat ← GPU kernel lookup table
├── lib/
│   ├── librocblas.so.4.3.0                ← Actual shared library (2.1 GB)
│   ├── librocblas.so.4 → librocblas.so.4.3.0  ← Symlink
│   ├── librocblas.so → librocblas.so.4        ← Symlink
│   └── rocblas/
│       └── library/
│           ├── TensileLibrary_gfx1150.co          ← Compiled GPU kernels (code object)
│           ├── TensileLibrary_gfx1150_Kernels.so  ← Kernel implementations
│           └── TensileLibrary_fallback.dat        ← Fallback kernel data
├── include/
│   └── rocblas/
│       ├── rocblas.h                      ← Main header file
│       ├── rocblas-types.h                ← Type definitions
│       ├── rocblas-functions.h            ← Function declarations
│       └── internal/
│           └── rocblas-device-functions.h ← Device function headers
└── share/
    ├── doc/
    │   └── rocblas/
    │       └── README.md                  ← Documentation
    └── rocblas/
        └── cmake/
            ├── rocblas-config.cmake       ← CMake package config
            └── rocblas-targets.cmake      ← CMake target exports
```

**Why symlinks?** Standard library convention:
- `librocblas.so` - Used by compiler linker (`-lrocblas`)
- `librocblas.so.4` - SONAME (embedded in binaries)
- `librocblas.so.4.3.0` - Actual file with full version

#### Step 3: Artifact Descriptor (Component Slicing)

Now we need to split this stage directory into components. A separate file defines the rules.

**File:** `math-libs/BLAS/artifact-blas.toml`

```toml
# This file defines how to slice the rocBLAS stage directory into components

# LIB component = runtime files needed to use the library
[components.lib."math-libs/BLAS/rocBLAS/stage"]
include = [
  "bin/rocblas/library/**",    # GPU kernel data
  "lib/rocblas/library/**",    # Compiled kernels
]
# Note: lib/*.so files are included by default pattern matching

# DEV component = development files needed to build against the library
[components.dev."math-libs/BLAS/rocBLAS/stage"]
# Uses built-in defaults:
#   include/**: All headers
#   share/**/cmake/**: All CMake configs
#   lib/**/*.a: Static libraries (if any)

# TEST component = test executables and data
[components.test."math-libs/BLAS/rocBLAS/stage"]
include = [
  "bin/rocblas-bench*",
  "bin/rocblas-test*",
  "bin/rocblas_gentest.py",
  "bin/rocblas_gtest.data",
]

# DOC component = documentation
[components.doc."math-libs/BLAS/rocBLAS/stage"]
# Uses built-in defaults:
#   share/doc/**: All documentation

# DBG component = debug symbols
[components.dbg."math-libs/BLAS/rocBLAS/stage"]
# Uses built-in defaults:
#   .build-id/**/*.debug: Debug symbol files
```

**Built-in pattern defaults** (from `build_tools/_therock_utils/artifact_builder.py`):

```python
# lib component automatically includes
ComponentDefaults("lib", includes=[
    "**/*.so",      # Shared libraries
    "**/*.so.*",    # Versioned shared libraries
    "**/*.dll",     # Windows DLLs
])

# dev component automatically includes
ComponentDefaults("dev", includes=[
    "**/*.a",           # Static libraries
    "**/*.lib",         # Windows static libraries
    "**/cmake/**",      # CMake package configs
    "**/include/**",    # Header files
    "**/pkgconfig/**",  # pkg-config files
], extends=["lib"])  # Also includes lib patterns
```

#### Step 4: Create Component Artifacts

**Command:**
```bash
ninja -C build artifact-blas
```

**What happens:** The artifact builder:
1. Reads `artifact-blas.toml`
2. Scans `build/math-libs/BLAS/rocBLAS/stage/`
3. Applies include/exclude patterns for each component
4. Creates separate directories for each component

**Output directories:**

```
build/artifacts/
├── blas_lib_gfx1150/
│   ├── artifact_manifest.txt          ← List of files in this component
│   ├── bin/
│   │   └── rocblas/
│   │       └── library/
│   │           └── TensileLibrary_gfx1150.dat
│   └── lib/
│       ├── librocblas.so.4.3.0
│       ├── librocblas.so.4 → librocblas.so.4.3.0
│       ├── librocblas.so → librocblas.so.4
│       └── rocblas/
│           └── library/
│               ├── TensileLibrary_gfx1150.co
│               ├── TensileLibrary_gfx1150_Kernels.so
│               └── TensileLibrary_fallback.dat
│
├── blas_dev_gfx1150/
│   ├── artifact_manifest.txt
│   ├── include/
│   │   └── rocblas/
│   │       ├── rocblas.h
│   │       ├── rocblas-types.h
│   │       ├── rocblas-functions.h
│   │       └── internal/
│   │           └── rocblas-device-functions.h
│   └── share/
│       └── rocblas/
│           └── cmake/
│               ├── rocblas-config.cmake
│               └── rocblas-targets.cmake
│
├── blas_test_gfx1150/
│   ├── artifact_manifest.txt
│   └── bin/
│       ├── rocblas-bench
│       ├── rocblas-test
│       ├── rocblas_gentest.py
│       └── rocblas_gtest.data
│
└── blas_doc_gfx1150/
    ├── artifact_manifest.txt
    └── share/
        └── doc/
            └── rocblas/
                └── README.md
```

**artifact_manifest.txt example:**
```
bin/rocblas/library/TensileLibrary_gfx1150.dat
lib/librocblas.so
lib/librocblas.so.4
lib/librocblas.so.4.3.0
lib/rocblas/library/TensileLibrary_gfx1150.co
lib/rocblas/library/TensileLibrary_gfx1150_Kernels.so
lib/rocblas/library/TensileLibrary_fallback.dat
```

#### Step 5: Compress into Archives

**Command:**
```bash
ninja -C build archive-blas
```

**What happens:** Each component directory gets compressed using `tar` with `xz` compression.

**Output archives:**
```
build/artifacts/
├── blas_lib_gfx1150.tar.xz       (2.1 GB compressed)
├── blas_dev_gfx1150.tar.xz       (500 KB compressed)
├── blas_test_gfx1150.tar.xz      (50 MB compressed)
└── blas_doc_gfx1150.tar.xz       (100 KB compressed)
```

**Why compress?** To upload to S3 and download faster in later CI stages.

#### Step 6: Upload to S3

**Command (in CI):**
```bash
aws s3 sync build/artifacts/ s3://therock-ci-artifacts/12345678-linux/
```

**S3 bucket structure:**
```
s3://therock-ci-artifacts/12345678-linux/
├── blas_lib_gfx1150.tar.xz
├── blas_lib_gfx94x.tar.xz
├── blas_lib_gfx950.tar.xz
├── blas_dev_gfx1150.tar.xz
├── blas_dev_gfx94x.tar.xz
├── blas_dev_gfx950.tar.xz
├── core-hip_lib_generic.tar.xz
├── core-hip_run_generic.tar.xz
├── core-hip_dev_generic.tar.xz
└── ... (100+ component artifacts)
```

**Who uses this?**
- Later CI stages (download as dependencies)
- Python packaging workflow (downloads to create wheels)
- RPM/DEB packaging workflow (downloads to create packages)

### Naming Convention

Format: `{artifact_name}_{component}_{target_family}.tar.xz`

Examples:
- `blas_lib_gfx1150.tar.xz` = artifact "blas", component "lib", target "gfx1150"
- `core-hip_lib_generic.tar.xz` = artifact "core-hip", component "lib", target "generic"
- `fft_dev_gfx94x.tar.xz` = artifact "fft", component "dev", target "gfx94x"

**"generic" vs GPU-specific:**
- `generic` = works on any GPU (host-side code like HIP runtime)
- `gfx1150` = only for RX 7000 series GPUs (contains compiled GPU kernels)
- `gfx94x` = only for MI300 GPUs
- `gfx950` = only for MI350 GPUs

---

<a name="part-4-python-packages"></a>
## Part 4: Creating Python Packages

### Overview

Python packaging takes the component artifacts and reorganizes them into Python wheels (`.whl` files) that can be installed with pip.

**Key transformations:**
1. Filter which artifacts go into which package
2. Remove symlinks (wheels don't support them)
3. Add Python wrapper code
4. Create wheel metadata

### The 4 Python Packages

1. **`rocm`** (selector)
   - Format: Source distribution (`.tar.gz`)
   - Contains: Pure Python code, no binaries
   - Purpose: Detects GPU and installs correct dependencies
   - Size: ~100 KB

2. **`rocm-sdk-core`** (runtime)
   - Format: Binary wheel (`.whl`)
   - Contains: HIP runtime, compiler tools
   - Purpose: Required for any GPU computation
   - Size: ~500 MB

3. **`rocm-sdk-libraries-{gpu}`** (math libraries)
   - Format: Binary wheel (`.whl`)
   - Contains: rocBLAS, rocFFT, rocRAND, etc.
   - Purpose: Optional, for math/ML workloads
   - Size: ~2.1 GB
   - **GPU-specific:** Different wheel for each GPU family

4. **`rocm-sdk-devel`** (development)
   - Format: Binary wheel (`.whl`)
   - Contains: Headers, CMake configs, symlinks
   - Purpose: Optional, for building ROCm applications
   - Size: ~300 MB

### Building Python Packages

#### Step 1: Download Component Artifacts

**Command:**
```bash
python ./build_tools/fetch_artifacts.py \
  --run-id=12345678 \
  --artifact-group=gfx1150 \
  --output-dir=./artifacts
```

**What happens:** Downloads and extracts artifacts from S3.

**Result:**
```
artifacts/
├── core-hip_lib_generic/
│   ├── artifact_manifest.txt
│   ├── lib/
│   │   ├── libamdhip64.so.6.2.0
│   │   ├── libamdhip64.so.6 → libamdhip64.so.6.2.0
│   │   └── libamdhip64.so → libamdhip64.so.6
│   └── share/hip/
│
├── core-hip_run_generic/
│   ├── artifact_manifest.txt
│   └── bin/
│       ├── hipcc
│       ├── hipconfig
│       └── hipify-perl
│
├── core-hip_dev_generic/
│   ├── artifact_manifest.txt
│   └── include/
│       └── hip/
│           ├── hip_runtime.h
│           └── hip_runtime_api.h
│
├── blas_lib_gfx1150/
│   ├── artifact_manifest.txt
│   └── lib/
│       ├── librocblas.so.4.3.0
│       ├── librocblas.so.4 → librocblas.so.4.3.0
│       └── librocblas.so → librocblas.so.4
│
└── blas_dev_gfx1150/
    ├── artifact_manifest.txt
    └── include/
        └── rocblas/
            └── rocblas.h
```

#### Step 2: Build Python Packages

**Command:**
```bash
python ./build_tools/build_python_packages.py \
  --artifact-dir=./artifacts \
  --dest-dir=./packages \
  --version=7.12.0.dev0
```

**What happens:** Script applies filters to select which artifacts go into each package.

#### Step 2a: Build rocm-sdk-core Package

**Filter logic:**

**File:** `build_tools/build_python_packages.py`
```python
def core_artifact_filter(an: ArtifactName) -> bool:
    """Determines if artifact goes into core package"""
    core = an.name in [
        "core-hip",        # HIP runtime (from BUILD_TOPOLOGY.toml)
        "core-runtime",    # ROCr runtime
        "base",            # Base ROCm tools
        "amd-llvm",        # LLVM compiler
        "hipify",          # Code conversion tool
        "core-amdsmi",     # System management
        "host-blas",       # CPU BLAS library
        "rocprofiler-sdk", # Profiling runtime
        "sysdeps",         # Bundled dependencies
    ] and an.component in ["lib", "run"]

    # Special case: HIP headers needed by hiprtc (runtime compilation)
    hip_dev = an.name in ["core-hip"] and an.component in ["dev"]

    return core or hip_dev
```

**Applied to our artifacts:**
- ✅ `core-hip_lib_generic` (name="core-hip", component="lib")
- ✅ `core-hip_run_generic` (name="core-hip", component="run")
- ✅ `core-hip_dev_generic` (name="core-hip", component="dev" - SPECIAL CASE)
- ❌ `blas_lib_gfx1150` (name="blas" not in core list)
- ❌ `blas_dev_gfx1150` (name="blas" not in core list)

**Package creation:**

```python
core = PopulatedDistPackage(params, logical_name="core")
core.populate_runtime_files(
    params.filter_artifacts(core_artifact_filter)
)
```

**Key transformation - symlink removal:**

Original artifact has:
```
lib/libamdhip64.so → libamdhip64.so.6 → libamdhip64.so.6.2.0
```

Python wheel will have:
```
lib/libamdhip64.so.6.2.0  ← Only the SONAME file (actual file)
```

Why? **Python wheels don't support symlinks.** So we:
1. Keep only the SONAME file (`libamdhip64.so.6.2.0`)
2. Remove all symlinks (`libamdhip64.so`, `libamdhip64.so.6`)
3. Application dynamic linker will find the SONAME file directly

**Created package structure:**
```
packages/rocm-sdk-core/
├── src/
│   └── rocm_sdk_core/
│       ├── __init__.py                 ← NEW: Python module
│       ├── _dist_info.py               ← NEW: Package metadata
│       └── platform/                   ← FROM: Artifacts
│           ├── bin/
│           │   ├── hipcc               ← From core-hip_run
│           │   ├── hipconfig           ← From core-hip_run
│           │   └── hipify-perl         ← From core-hip_run
│           ├── lib/
│           │   ├── libamdhip64.so.6.2.0    ← From core-hip_lib (SONAME only)
│           │   ├── libhiprtc.so.6.2.0      ← From core-hip_lib (SONAME only)
│           │   └── libamd_comgr.so.2.8.0   ← From core-hip_lib (SONAME only)
│           └── include/
│               └── hip/
│                   ├── hip_runtime.h        ← From core-hip_dev (SPECIAL)
│                   └── hip_runtime_api.h    ← From core-hip_dev (SPECIAL)
├── setup.py                            ← NEW: Python package setup
├── pyproject.toml                      ← NEW: Build system requirements
└── README.md                           ← NEW: Package description
```

**What's in `__init__.py`:**
```python
"""ROCm SDK Core Package"""
from pathlib import Path

# Export path to platform files
_PLATFORM_PATH = Path(__file__).parent / "platform"

def get_platform_path():
    """Returns path to ROCm installation"""
    return _PLATFORM_PATH

def get_lib_path():
    """Returns path to shared libraries"""
    return _PLATFORM_PATH / "lib"
```

#### Step 2b: Build rocm-sdk-libraries-gfx1150 Package

**Filter logic:**
```python
def libraries_artifact_filter(target_family: str, an: ArtifactName) -> bool:
    """Determines if artifact goes into libraries package"""
    return (
        an.name in [
            "blas",          # rocBLAS + hipBLAS
            "fft",           # rocFFT + hipFFT
            "rand",          # rocRAND + hipRAND
            "sparse",        # rocSPARSE + hipSPARSE
            "solver",        # rocSOLVER + hipSOLVER
            "rccl",          # RCCL (communication)
            "miopen",        # MIOpen (ML library)
            "hipdnn",        # hipDNN backend
        ]
        and an.component in ["lib"]  # Only lib component
        and (an.target_family == target_family or an.target_family == "generic")
    )

lib = PopulatedDistPackage(params, logical_name="libraries", target_family="gfx1150")
lib.populate_runtime_files(
    params.filter_artifacts(
        filter=functools.partial(libraries_artifact_filter, "gfx1150")
    )
)
```

**Applied to our artifacts:**
- ❌ `core-hip_lib_generic` (name="core-hip" not in libraries list)
- ✅ `blas_lib_gfx1150` (name="blas", component="lib", family="gfx1150")
- ❌ `blas_dev_gfx1150` (component="dev", not "lib")

**Created package structure:**
```
packages/rocm-sdk-libraries/
├── src/
│   └── rocm_sdk_libraries/
│       ├── __init__.py
│       ├── _dist_info.py
│       └── platform/
│           ├── bin/
│           │   └── rocblas/
│           │       └── library/
│           │           └── TensileLibrary_gfx1150.dat
│           └── lib/
│               ├── librocblas.so.4.3.0     ← SONAME only (symlinks removed)
│               ├── libhipblas.so.2.3.0     ← SONAME only
│               ├── librocfft.so.1.0.28     ← SONAME only
│               └── rocblas/
│                   └── library/
│                       ├── TensileLibrary_gfx1150.co
│                       └── TensileLibrary_gfx1150_Kernels.so
└── setup.py
```

#### Step 2c: Build rocm-sdk-devel Package

**Different approach - includes everything not in runtime packages:**

```python
devel = PopulatedDistPackage(params, logical_name="devel")
devel.populate_devel_files(
    addl_artifact_names=[
        "prim",           # rocPRIM (header-only library)
        "rocwmma",        # rocWMMA (header-only library)
        "flatbuffers",    # Third-party dependency
        "nlohmann-json",  # Third-party dependency
    ],
    tarball_compression=True
)
```

**What populate_devel_files() does:**

1. **Includes all `dev` components** from artifacts that were used in runtime packages
2. **Recreates symlinks** to files in runtime packages (NO file duplication!)
3. **Includes header-only libraries** that have no runtime component
4. **Stores everything in `_devel.tar.xz`** (wheels can't contain symlinks directly)

**Created package structure:**
```
packages/rocm-sdk-devel/
├── src/
│   └── rocm_sdk_devel/
│       ├── __init__.py
│       ├── _dist_info.py
│       └── _devel.tar.xz          ← SPECIAL: Tarball inside wheel!
│           │
│           └── (when extracted contains:)
│               │
│               ├── include/
│               │   ├── rocblas/
│               │   │   ├── rocblas.h
│               │   │   ├── rocblas-types.h
│               │   │   └── rocblas-functions.h
│               │   ├── hip/
│               │   │   └── (additional headers not in core)
│               │   ├── rocprim/         ← Header-only library
│               │   │   └── rocprim.hpp
│               │   └── rocwmma/         ← Header-only library
│               │       └── rocwmma.hpp
│               │
│               ├── lib/
│               │   │
│               │   │ These are SYMLINKS (point to runtime packages):
│               │   │
│               │   ├── libamdhip64.so → ../../../_rocm_sdk_core_7_12_0_dev0_*/platform/lib/libamdhip64.so.6.2.0
│               │   ├── libamdhip64.so.6 → ../../../_rocm_sdk_core_7_12_0_dev0_*/platform/lib/libamdhip64.so.6.2.0
│               │   ├── librocblas.so → ../../../_rocm_sdk_libraries_gfx1150_7_12_0_dev0_*/platform/lib/librocblas.so.4.3.0
│               │   └── librocblas.so.4 → ../../../_rocm_sdk_libraries_gfx1150_7_12_0_dev0_*/platform/lib/librocblas.so.4.3.0
│               │
│               └── share/
│                   ├── hip/
│                   │   └── cmake/
│                   │       ├── hip-config.cmake
│                   │       └── hip-targets.cmake
│                   └── rocblas/
│                       └── cmake/
│                           ├── rocblas-config.cmake
│                           └── rocblas-targets.cmake
└── setup.py
```

**CRITICAL:** The symlinks use **relative paths with wildcards** (the `*`). The wildcard gets filled in during package build to match the installed package name.

#### Step 2d: Build rocm Selector Package

**Pure Python only - no binaries:**

```python
PopulatedDistPackage(params, logical_name="meta")
```

**Created package structure:**
```
packages/rocm/
├── src/
│   └── rocm_sdk/
│       ├── __init__.py
│       ├── _dist_info.py          ← GPU detection code
│       ├── cli.py                 ← rocm-sdk command implementation
│       └── test.py                ← Self-test functionality
├── setup.py                       ← Dynamic dependency resolution
├── pyproject.toml
└── README.md
```

**What's in `_dist_info.py`:**
```python
"""Package metadata and GPU detection"""
import os
import subprocess

def discover_current_target_family() -> str | None:
    """Uses offload-arch tool to detect GPU"""
    result = subprocess.run(
        ["offload-arch", "--targets"],
        capture_output=True,
        text=True,
        timeout=5
    )
    if result.returncode == 0:
        targets = result.stdout.strip().split("\n")
        if targets:
            # Map gfx1150 → gfx1150, gfx942 → gfx94x, etc.
            return map_gfx_to_family(targets[0])
    return None

def determine_target_family() -> str:
    """Determines GPU in priority order"""
    # 1. Environment variable
    target = os.getenv("ROCM_SDK_TARGET_FAMILY")
    if target:
        return target

    # 2. Auto-detect
    target = discover_current_target_family()
    if target:
        return target

    # 3. Default (from package build)
    return DEFAULT_TARGET_FAMILY

# Package definitions
ALL_PACKAGES = {
    "core": PackageEntry("core", "rocm-sdk-core"),
    "libraries": PackageEntry("libraries", "rocm-sdk-libraries-{target_family}"),
    "devel": PackageEntry("devel", "rocm-sdk-devel"),
}

# Filled in at package build time
__version__ = "7.12.0.dev0"
DEFAULT_TARGET_FAMILY = "gfx1150"
AVAILABLE_TARGET_FAMILIES = ["gfx94x_dcgpu", "gfx950", "gfx110x_all", "gfx1150"]
```

**What's in `setup.py`:**
```python
"""Setup.py for rocm selector package"""
from pathlib import Path
import sys
from setuptools import setup

# Load _dist_info module
dist_info_path = Path(__file__).parent / "src" / "rocm_sdk" / "_dist_info.py"
dist_info_globals = {}
exec(dist_info_path.read_text(), dist_info_globals)

# Detect GPU at install time
determine_target_family = dist_info_globals["determine_target_family"]
detected_gpu = determine_target_family()

print(f"Detected GPU family: {detected_gpu}", file=sys.stderr)

# Build dependencies
ALL_PACKAGES = dist_info_globals["ALL_PACKAGES"]
version = dist_info_globals["__version__"]

install_requires = [
    f"rocm-sdk-core=={version}"  # Core always required
]

extras_require = {
    "libraries": [
        f"rocm-sdk-libraries-{detected_gpu}=={version}"
    ],
    "devel": [
        f"rocm-sdk-devel=={version}"
    ],
}

setup(
    name="rocm",
    version=version,
    install_requires=install_requires,
    extras_require=extras_require,
    entry_points={
        "console_scripts": [
            "rocm-sdk=rocm_sdk.cli:main",
        ],
    },
    # ... metadata
)
```

#### Step 3: Build Wheels

**Commands (called internally):**
```bash
python -m build --wheel ./packages/rocm-sdk-core
python -m build --wheel ./packages/rocm-sdk-libraries
python -m build --wheel ./packages/rocm-sdk-devel
python -m build --sdist ./packages/rocm
```

**Output:**
```
packages/dist/
├── rocm-7.12.0.dev0.tar.gz                                           (100 KB)
├── rocm_sdk_core-7.12.0.dev0-py3-none-linux_x86_64.whl              (500 MB)
├── rocm_sdk_libraries_gfx1150-7.12.0.dev0-py3-none-linux_x86_64.whl (2.1 GB)
└── rocm_sdk_devel-7.12.0.dev0-py3-none-linux_x86_64.whl             (300 MB)
```

**Wheel naming format:**
```
{package}-{version}-{python_tag}-{abi_tag}-{platform_tag}.whl

rocm_sdk_libraries_gfx1150-7.12.0.dev0-py3-none-linux_x86_64.whl
└─────┬─────────┘└─────┬─┘ └─┬──────┘ └─┬┘ └─┬─┘ └──────┬──────┘
      │                │      │           │    │          └─ Platform: Linux x86_64
      │                │      │           │    └─ ABI: none (no C extensions)
      │                │      │           └─ Python: py3 (any Python 3.x)
      │                │      └─ Nightly build on March 4, 2026
      │                └─ Version: 7.12.0 development build
      └─ Package name with GPU family
```

#### Step 4: Upload to S3 PyPI Index

**Script creates pip-compatible index:**
```bash
python ./third-party/indexer/indexer.py \
  packages/dist/ \
  --filter "*.whl" "*.tar.gz" \
  --output packages/dist/index.html
```

**Generated index.html:**
```html
<!DOCTYPE html>
<html>
<head><title>ROCm Python Packages - gfx1150</title></head>
<body>
<h1>ROCm Python Packages</h1>
<a href="rocm-7.12.0.dev0.tar.gz#sha256=abc123...">rocm-7.12.0.dev0.tar.gz</a><br/>
<a href="rocm_sdk_core-7.12.0.dev0-py3-none-linux_x86_64.whl#sha256=def456...">rocm_sdk_core-7.12.0.dev0-py3-none-linux_x86_64.whl</a><br/>
<a href="rocm_sdk_libraries_gfx1150-7.12.0.dev0-py3-none-linux_x86_64.whl#sha256=ghi789...">rocm_sdk_libraries_gfx1150-7.12.0.dev0-py3-none-linux_x86_64.whl</a><br/>
<a href="rocm_sdk_devel-7.12.0.dev0-py3-none-linux_x86_64.whl#sha256=jkl012...">rocm_sdk_devel-7.12.0.dev0-py3-none-linux_x86_64.whl</a><br/>
</body>
</html>
```

**Upload to S3:**
```bash
aws s3 sync packages/dist/ s3://therock-nightly-python/v2/gfx1150/
```

**S3 structure:**
```
s3://therock-nightly-python/v2/
├── gfx1150/
│   ├── index.html
│   ├── rocm-7.12.0.dev0.tar.gz
│   ├── rocm_sdk_core-7.12.0.dev0-py3-none-linux_x86_64.whl
│   ├── rocm_sdk_libraries_gfx1150-7.12.0.dev0-py3-none-linux_x86_64.whl
│   └── rocm_sdk_devel-7.12.0.dev0-py3-none-linux_x86_64.whl
├── gfx94X-dcgpu/
│   ├── index.html
│   ├── rocm-7.12.0.dev0.tar.gz
│   ├── rocm_sdk_core-7.12.0.dev0-py3-none-linux_x86_64.whl
│   ├── rocm_sdk_libraries_gfx94x_dcgpu-7.12.0.dev0-py3-none-linux_x86_64.whl
│   └── rocm_sdk_devel-7.12.0.dev0-py3-none-linux_x86_64.whl
└── gfx950/
    ├── index.html
    └── ... (similar structure)
```

**Public URLs:**
- `https://rocm.nightlies.amd.com/v2/gfx1150/`
- `https://rocm.nightlies.amd.com/v2/gfx94X-dcgpu/`
- `https://rocm.nightlies.amd.com/v2/gfx950/`

---

<a name="part-5-pip-install"></a>
## Part 5: User Installation with pip

Now a user on their machine wants to install ROCm.

### User's Command

```bash
python -m venv .venv
source .venv/bin/activate
pip install rocm[libraries,devel] --pre \
  --extra-index-url https://rocm.nightlies.amd.com/v2/gfx1150/
```

**Breaking down the command:**
- `python -m venv .venv` - Create virtual environment
- `source .venv/bin/activate` - Activate it
- `pip install rocm[libraries,devel]` - Install rocm with extras
  - `rocm` - Package name
  - `[libraries,devel]` - Extras to install
- `--pre` - Allow pre-release versions (development builds)
- `--extra-index-url https://...` - Look here for packages

### pip Processing Steps

#### Phase 1: Find the Package

pip searches for `rocm`:
1. Default index: `https://pypi.org/simple/rocm/` (not found)
2. Extra index: `https://rocm.nightlies.amd.com/v2/gfx1150/` (found!)

Downloads `index.html`, finds available packages.

#### Phase 2: Download rocm Package

pip downloads:
```
GET https://rocm.nightlies.amd.com/v2/gfx1150/rocm-7.12.0.dev0.tar.gz
```

Extracts to temporary directory:
```
/tmp/pip-install-xyz/rocm-7.12.0.dev0/
├── src/rocm_sdk/
│   ├── __init__.py
│   ├── _dist_info.py
│   └── cli.py
├── setup.py
├── pyproject.toml
└── PKG-INFO
```

#### Phase 3: Run setup.py (GPU Detection Happens Here!)

pip runs `python setup.py --metadata` to get package info.

**setup.py executes:**

1. Loads `_dist_info.py`:
```python
exec(dist_info_path.read_text(), dist_info_globals)
```

2. Calls GPU detection:
```python
determine_target_family = dist_info_globals["determine_target_family"]
detected_gpu = determine_target_family()
```

3. `determine_target_family()` runs:
```python
def determine_target_family() -> str:
    # Priority 1: Check environment variable
    target = os.getenv("ROCM_SDK_TARGET_FAMILY")
    if target:
        return target

    # Priority 2: Run offload-arch tool
    result = subprocess.run(
        ["offload-arch", "--targets"],
        capture_output=True,
        text=True
    )
    if result.returncode == 0:
        # Parse output: "gfx1150\n"
        targets = result.stdout.strip().split("\n")
        if targets:
            # gfx1150 → gfx1150
            # gfx942 → gfx94x_dcgpu
            # gfx950 → gfx950
            return map_gfx_to_family(targets[0])

    # Priority 3: Use default from package
    return DEFAULT_TARGET_FAMILY  # "gfx1150"
```

**On user's machine with RX 7900 XTX:**
```bash
$ offload-arch --targets
gfx1150
```

Returns: `detected_gpu = "gfx1150"`

4. Builds dependency list:
```python
version = "7.12.0.dev0"

install_requires = ["rocm-sdk-core==7.12.0.dev0"]

extras_require = {
    "libraries": [f"rocm-sdk-libraries-{detected_gpu}==7.12.0.dev0"],
    # Expands to: "rocm-sdk-libraries-gfx1150==7.12.0.dev0"

    "devel": ["rocm-sdk-devel==7.12.0.dev0"],
}

setup(
    name="rocm",
    version=version,
    install_requires=install_requires,
    extras_require=extras_require,
)
```

**pip sees this output:**
```
Detected GPU family: gfx1150
Installing rocm[libraries,devel] requires:
  - rocm-sdk-core==7.12.0.dev0
  - rocm-sdk-libraries-gfx1150==7.12.0.dev0
  - rocm-sdk-devel==7.12.0.dev0
```

#### Phase 4: Resolve Dependencies

pip creates full dependency tree:

```
rocm==7.12.0.dev0
├── install_requires:
│   └── rocm-sdk-core==7.12.0.dev0
├── extras["libraries"]:
│   └── rocm-sdk-libraries-gfx1150==7.12.0.dev0
└── extras["devel"]:
    └── rocm-sdk-devel==7.12.0.dev0
```

**Flattened list:**
1. `rocm==7.12.0.dev0`
2. `rocm-sdk-core==7.12.0.dev0`
3. `rocm-sdk-libraries-gfx1150==7.12.0.dev0`
4. `rocm-sdk-devel==7.12.0.dev0`

#### Phase 5: Download All Packages

pip downloads from extra index:

```
Collecting rocm==7.12.0.dev0
  Downloading rocm-7.12.0.dev0.tar.gz (100 kB)

Collecting rocm-sdk-core==7.12.0.dev0
  Downloading rocm_sdk_core-7.12.0.dev0-py3-none-linux_x86_64.whl (500.0 MB)
  ████████████████████████████████ 500.0/500.0 MB 50.0 MB/s eta 0:00:00

Collecting rocm-sdk-libraries-gfx1150==7.12.0.dev0
  Downloading rocm_sdk_libraries_gfx1150-7.12.0.dev0-py3-none-linux_x86_64.whl (2.1 GB)
  ████████████████████████████████ 2.1/2.1 GB 50.0 MB/s eta 0:00:00

Collecting rocm-sdk-devel==7.12.0.dev0
  Downloading rocm_sdk_devel-7.12.0.dev0-py3-none-linux_x86_64.whl (300.0 MB)
  ████████████████████████████████ 300.0/300.0 MB 50.0 MB/s eta 0:00:00
```

Total download: ~3 GB

#### Phase 6: Install Packages

pip extracts wheels to site-packages:

```
.venv/lib/python3.12/site-packages/

├── rocm_sdk/
│   ├── __init__.py
│   ├── _dist_info.py
│   ├── cli.py
│   └── test.py
│
├── rocm-7.12.0.dev0.dist-info/
│   ├── METADATA
│   ├── RECORD
│   ├── entry_points.txt           ← Defines rocm-sdk command
│   └── top_level.txt
│
├── _rocm_sdk_core_7_12_0_dev0_123abc/     ← Note the nonce suffix
│   └── platform/
│       ├── bin/
│       │   ├── hipcc
│       │   ├── hipconfig
│       │   └── hipify-perl
│       ├── lib/
│       │   ├── libamdhip64.so.6.2.0       ← Actual file (150 MB)
│       │   ├── libhiprtc.so.6.2.0         ← Actual file (80 MB)
│       │   └── libamd_comgr.so.2.8.0      ← Actual file (120 MB)
│       └── include/
│           └── hip/
│               ├── hip_runtime.h
│               └── hip_runtime_api.h
│
├── rocm_sdk_core/                         ← Symlink to real location
│   → _rocm_sdk_core_7_12_0_dev0_123abc
│
├── rocm_sdk_core-7.12.0.dev0.dist-info/
│   ├── METADATA
│   └── RECORD
│
├── _rocm_sdk_libraries_gfx1150_7_12_0_dev0_456def/
│   └── platform/
│       ├── bin/
│       │   └── rocblas/library/
│       │       └── TensileLibrary_gfx1150.dat
│       └── lib/
│           ├── librocblas.so.4.3.0        ← Actual file (2.1 GB)
│           ├── libhipblas.so.2.3.0        ← Actual file (50 MB)
│           └── rocblas/library/
│               ├── TensileLibrary_gfx1150.co
│               └── TensileLibrary_gfx1150_Kernels.so
│
├── rocm_sdk_libraries/
│   → _rocm_sdk_libraries_gfx1150_7_12_0_dev0_456def
│
├── rocm_sdk_libraries-7.12.0.dev0.dist-info/
│   ├── METADATA
│   └── RECORD
│
├── _rocm_sdk_devel_7_12_0_dev0_789ghi/
│   ├── __init__.py
│   ├── _dist_info.py
│   └── _devel.tar.xz                      ← Not extracted yet
│
├── rocm_sdk_devel/
│   → _rocm_sdk_devel_7_12_0_dev0_789ghi
│
└── rocm_sdk_devel-7.12.0.dev0.dist-info/
    ├── METADATA
    └── RECORD
```

**Why the nonce suffix?** (`_7_12_0_dev0_123abc`)
- Allows multiple versions to coexist
- Prevents import conflicts
- Nonce = version + random hash

#### Phase 7: Create Command-Line Tool

From `rocm` package's `entry_points.txt`:
```
[console_scripts]
rocm-sdk = rocm_sdk.cli:main
```

pip creates executable:
```
.venv/bin/rocm-sdk
```

**Content:**
```python
#!/path/to/.venv/bin/python
# -*- coding: utf-8 -*-
import re
import sys
from rocm_sdk.cli import main
if __name__ == '__main__':
    sys.exit(main())
```

User can now run:
```bash
$ rocm-sdk version
7.12.0.dev0

$ rocm-sdk info
ROCm SDK Information:
  Version: 7.12.0.dev0
  Target Family: gfx1150
  Installed Packages:
    - rocm-sdk-core (required)
    - rocm-sdk-libraries-gfx1150 (optional)
    - rocm-sdk-devel (optional)
```

#### Phase 8: Extract Devel Tarball (On First Access)

The `_devel.tar.xz` is **NOT** extracted during installation. It's extracted on first use.

**When user runs:**
```bash
cmake -B build
```

CMake looks for `rocblas-config.cmake`. This triggers:

```python
# In rocm_sdk_devel/__init__.py
def get_devel_path():
    """Returns path to devel files, extracting if needed"""
    devel_dir = Path(__file__).parent / "platform"

    if not devel_dir.exists():
        # Extract _devel.tar.xz
        tarball = Path(__file__).parent / "_devel.tar.xz"
        with tarfile.open(tarball, "r:xz") as tf:
            tf.extractall(Path(__file__).parent)

    return devel_dir
```

**After extraction:**
```
_rocm_sdk_devel_7_12_0_dev0_789ghi/
├── _devel.tar.xz                          ← Original (kept)
└── platform/                              ← Extracted
    ├── include/
    │   ├── rocblas/
    │   │   ├── rocblas.h
    │   │   ├── rocblas-types.h
    │   │   └── rocblas-functions.h
    │   ├── hip/
    │   │   └── (additional headers)
    │   ├── rocprim/
    │   │   └── rocprim.hpp
    │   └── rocwmma/
    │       └── rocwmma.hpp
    ├── lib/
    │   ├── libamdhip64.so → ../../_rocm_sdk_core_7_12_0_dev0_123abc/platform/lib/libamdhip64.so.6.2.0
    │   ├── libamdhip64.so.6 → ../../_rocm_sdk_core_7_12_0_dev0_123abc/platform/lib/libamdhip64.so.6.2.0
    │   ├── librocblas.so → ../../_rocm_sdk_libraries_gfx1150_7_12_0_dev0_456def/platform/lib/librocblas.so.4.3.0
    │   └── librocblas.so.4 → ../../_rocm_sdk_libraries_gfx1150_7_12_0_dev0_456def/platform/lib/librocblas.so.4.3.0
    └── share/
        ├── hip/cmake/
        │   ├── hip-config.cmake
        │   └── hip-targets.cmake
        └── rocblas/cmake/
            ├── rocblas-config.cmake
            └── rocblas-targets.cmake
```

**Symlinks now work!**

Example symlink resolution:
```
_rocm_sdk_devel_*/platform/lib/librocblas.so
    ↓ (symlink points to)
../../_rocm_sdk_libraries_gfx1150_*/platform/lib/librocblas.so.4.3.0
    ↓ (resolves to)
/path/to/.venv/lib/python3.12/site-packages/_rocm_sdk_libraries_gfx1150_7_12_0_dev0_456def/platform/lib/librocblas.so.4.3.0
```

---

<a name="part-6-storage"></a>
## Part 6: Storage Efficiency and Symlinks

### The Question

**Q:** If I install `rocm[libraries,devel]`, do I get `librocblas.so.4.3.0` stored twice?

**A:** **NO.** You get it **once** in the libraries package. The devel package creates **symlinks** pointing to it.

### Proof

Run this command after installation:

```bash
find .venv/lib/python3.12/site-packages -name "librocblas.so*" -ls
```

**Output:**
```
2148532  2097152 -rw-r--r-- 1 user user 2147483648 Mar  4 12:00 .venv/lib/python3.12/site-packages/_rocm_sdk_libraries_gfx1150_7_12_0_dev0_456def/platform/lib/librocblas.so.4.3.0
2148533        0 lrwxrwxrwx 1 user user        112 Mar  4 12:00 .venv/lib/python3.12/site-packages/_rocm_sdk_devel_7_12_0_dev0_789ghi/platform/lib/librocblas.so -> ../../_rocm_sdk_libraries_gfx1150_7_12_0_dev0_456def/platform/lib/librocblas.so.4.3.0
2148534        0 lrwxrwxrwx 1 user user        112 Mar  4 12:00 .venv/lib/python3.12/site-packages/_rocm_sdk_devel_7_12_0_dev0_789ghi/platform/lib/librocblas.so.4 -> ../../_rocm_sdk_libraries_gfx1150_7_12_0_dev0_456def/platform/lib/librocblas.so.4.3.0
```

**Breakdown:**
- **2,097,152 KB (2.1 GB)** - Actual file in libraries package
- **0 bytes** - Symlink in devel package (points to actual file)
- **0 bytes** - Another symlink in devel package

### Storage Calculation

If you install `rocm[libraries,devel]`:

| Package | Contains | Actual Storage |
|---------|----------|----------------|
| `rocm` | Python code only | ~100 KB |
| `rocm-sdk-core` | HIP runtime binaries | ~500 MB |
| `rocm-sdk-libraries-gfx1150` | Math library binaries | ~2.1 GB |
| `rocm-sdk-devel` | Headers (300 MB) + symlinks (0 MB) | ~300 MB |
| **Total** | | **~2.9 GB** |

**NOT ~5.1 GB** (which it would be if libraries were duplicated in devel)

### How Symlinks Are Created

During package build, devel package remembers where each file was installed:

**Code:** `build_tools/_therock_utils/py_packaging.py`

```python
def _populate_devel_file(self, relpath: str, dest_path: Path, src_entry):
    # Check if this file was already included in a runtime package
    if self.params.files.has(relpath):
        # Yes! Create symlink instead of copying
        populated_package, populated_path = self.params.files.materialized_relpaths[relpath]

        # Calculate relative path from devel package to runtime package
        # Example: lib/librocblas.so.4.3.0
        # In devel: _rocm_sdk_devel_*/platform/lib/librocblas.so.4.3.0
        # Points to: _rocm_sdk_libraries_*/platform/lib/librocblas.so.4.3.0
        # Relative: ../../../_rocm_sdk_libraries_*/platform/lib/librocblas.so.4.3.0

        relpath_parts = Path(relpath).parts  # ["lib", "librocblas.so.4.3.0"]
        num_parts = len(relpath_parts)      # 2

        backtrack = Path(*([".."] * num_parts))  # ../../
        link_target = backtrack / populated_package.platform_dir.name / relpath

        # Create symlink
        dest_path.symlink_to(link_target)
```

**Example calculation:**

For file `lib/librocblas.so.4.3.0`:

1. **relpath parts:** `["lib", "librocblas.so.4.3.0"]` (2 parts)
2. **Backtrack:** Go up 2 levels: `../../`
3. **Target package:** `_rocm_sdk_libraries_gfx1150_*/platform`
4. **Final symlink:** `../../_rocm_sdk_libraries_gfx1150_*/platform/lib/librocblas.so.4.3.0`

When installed:
```
Symlink location:
  _rocm_sdk_devel_*/platform/lib/librocblas.so.4.3.0
Points to:
  ../../_rocm_sdk_libraries_gfx1150_*/platform/lib/librocblas.so.4.3.0
Resolves to:
  _rocm_sdk_libraries_gfx1150_*/platform/lib/librocblas.so.4.3.0
```

Perfect!

---

<a name="appendix-complete-example"></a>
## Appendix: Complete rocBLAS Example

This section traces **one library (rocBLAS)** through the entire pipeline from source code to pip installation, showing every step.

### Step 0: BUILD_TOPOLOGY.toml Definition

**File:** `/BUILD_TOPOLOGY.toml`

```toml
[artifacts.blas]
artifact_group = "math-libs"
type = "target-specific"
artifact_deps = ["core-hip", "rocprofiler-sdk"]
feature_group = "MATH_LIBS"
```

### Step 1: Source Code

**File:** `math-libs/rocm-libraries/rocBLAS/library/src/blas1/rocblas_axpy.cpp`

### Step 2: CMake Build

**Command:**
```bash
cmake -B build -GNinja -DTHEROCK_AMDGPU_FAMILIES=gfx1150
ninja -C build
```

**Output:**
```
build/math-libs/BLAS/rocBLAS/stage/lib/librocblas.so.4.3.0
```

### Step 3: Artifact Descriptor

**File:** `math-libs/BLAS/artifact-blas.toml`

```toml
[components.lib."math-libs/BLAS/rocBLAS/stage"]
include = ["lib/rocblas/**"]
```

### Step 4: Create Component Artifact

**Command:**
```bash
ninja -C build artifact-blas
```

**Output:**
```
build/artifacts/blas_lib_gfx1150.tar.xz (2.1 GB)
```

### Step 5: Upload to S3

**Command:**
```bash
aws s3 cp build/artifacts/blas_lib_gfx1150.tar.xz \
  s3://therock-ci-artifacts/12345678-linux/
```

### Step 6: Download for Python Packaging

**Command:**
```bash
python ./build_tools/fetch_artifacts.py \
  --run-id=12345678 \
  --output-dir=./artifacts
```

**Downloads:**
```
artifacts/blas_lib_gfx1150/lib/librocblas.so.4.3.0
```

### Step 7: Python Filter Matches

**Code:**
```python
an.name == "blas"  # From BUILD_TOPOLOGY.toml
an.component == "lib"
an.target_family == "gfx1150"
# ✅ Included in rocm-sdk-libraries-gfx1150
```

### Step 8: Create Wheel

**Command:**
```bash
python ./build_tools/build_python_packages.py ...
python -m build --wheel ./packages/rocm-sdk-libraries
```

**Output:**
```
rocm_sdk_libraries_gfx1150-7.12.0.dev0.whl
└── platform/lib/librocblas.so.4.3.0
```

### Step 9: Upload to S3

**Command:**
```bash
aws s3 cp rocm_sdk_libraries_gfx1150-*.whl \
  s3://therock-nightly-python/v2/gfx1150/
```

### Step 10: User Installs

**Command:**
```bash
pip install rocm[libraries]
```

**setup.py detects:**
```python
offload-arch --targets  # Returns: gfx1150
```

### Step 11: pip Downloads

```bash
pip downloads: rocm_sdk_libraries_gfx1150-7.12.0.dev0.whl
```

### Step 12: pip Installs

```
.venv/lib/python3.12/site-packages/
└── _rocm_sdk_libraries_gfx1150_*/platform/lib/librocblas.so.4.3.0
```

### Step 13: User Runs PyTorch

```python
import torch
a = torch.randn(1000, 1000, device='cuda')
b = torch.randn(1000, 1000, device='cuda')
c = torch.matmul(a, b)  # Uses librocblas.so.4.3.0
```

**Complete journey:** C++ source → component artifact → Python wheel → pip install → used by PyTorch

---

## Summary

### Key Concepts

1. **BUILD_TOPOLOGY.toml** is the master blueprint
   - Defines all artifacts and dependencies
   - Drives CMake, CI, and packaging
   - Single source of truth

2. **Component artifacts** are the reusable intermediate format
   - Built once in CI
   - Reused for Python, RPM, DEB packaging
   - Stored in S3

3. **Python packaging** reorganizes components
   - Filters artifacts by name (from topology)
   - Removes symlinks (wheel limitation)
   - Adds Python wrapper code

4. **pip installation** is dynamic
   - Detects GPU at install time
   - Resolves correct dependencies automatically
   - No file duplication (devel uses symlinks)

5. **Everything flows from BUILD_TOPOLOGY.toml**
   - Artifact names → Python package filters
   - Dependencies → Build order
   - Types (target-specific) → GPU-specific builds

---

**End of Document**

This is a complete, self-contained guide. You don't need to read any other documentation to understand how TheRock Python packaging works.
