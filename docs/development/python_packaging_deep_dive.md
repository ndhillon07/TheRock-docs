# ROCm Python Packaging: Complete Deep Dive
## From Source Code to pip Installation

This document provides a comprehensive, end-to-end explanation of how ROCm is built from source code, packaged into Python wheels, and installed via pip. It covers the entire pipeline with complete examples showing actual file structures and configurations.

---

## Table of Contents

1. [Overview: The Complete Journey](#overview)
2. [BUILD_TOPOLOGY.toml: The Master Orchestration](#build-topology)
3. [Stage 1: Source Code to Portable Linux Artifacts](#stage-1)
4. [Stage 2: Portable Artifacts to Python Packages](#stage-2)
5. [Stage 3: pip Installation and Dependency Resolution](#stage-3)
6. [Library Duplication: Storage Efficiency](#library-duplication)
7. [Advanced: How the rocm Selector Works](#selector-mechanism)
8. [Complete Example: rocBLAS End-to-End](#complete-example)

---

<a name="overview"></a>
## 1. Overview: The Complete Journey

### The Big Picture

```
┌─────────────────────────────────────────────────────────┐
│  STAGE 0: BUILD TOPOLOGY (Master Configuration)         │
│  Input: BUILD_TOPOLOGY.toml                             │
│  Output: CMake variables, CI matrix, feature flags      │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│  STAGE 1: BUILD PROCESS                                 │
│  Input: Source code in git repositories                 │
│  Output: Component artifacts (tar.xz files)             │
│  Time: ~4-8 hours                                       │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│  STAGE 2: PYTHON PACKAGING                              │
│  Input: Component artifacts from Stage 1                │
│  Output: Python wheels (.whl) and sdist (.tar.gz)       │
│  Time: ~10-20 minutes                                   │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│  STAGE 3: USER INSTALLATION                             │
│  Input: Python packages from Stage 2                    │
│  Output: Installed ROCm in Python environment           │
│  Time: ~2-5 minutes                                     │
└─────────────────────────────────────────────────────────┘
```

### What Gets Reused?

- **Stage 1 → Stage 2**: Component artifact files are reused directly (no rebuild)
- **Stage 2 → Stage 3**: Wheel files are downloaded and extracted
- **Between packages**: Devel package creates symlinks to runtime packages (NO file duplication!)

---

<a name="build-topology"></a>
## 2. BUILD_TOPOLOGY.toml: The Master Orchestration

### What is BUILD_TOPOLOGY.toml?

**`BUILD_TOPOLOGY.toml` is the single source of truth** for TheRock's entire build structure. It's the master configuration file that controls:

1. **What** gets built (artifacts)
2. **How** they're grouped (artifact groups)
3. **When** they're built (build stages)
4. **Which code** is needed (source sets)
5. **Dependencies** between everything

**Location:** `/BUILD_TOPOLOGY.toml` (root of repository)

### The Hierarchy

```
SOURCE SETS (git submodules needed)
    ↓ defines code needed for
BUILD STAGES (CI/CD pipeline jobs)
    ↓ builds groups of
ARTIFACT GROUPS (logical collections)
    ↓ contains
ARTIFACTS (individual build outputs)
    ↓ sliced into
COMPONENTS (lib, dev, doc, test, dbg)
    ↓ packaged into
PYTHON/RPM/DEB PACKAGES
```

### Example: Math Libraries

#### Source Set Definition

```toml
[source_sets.rocm-libraries]
description = "ROCm libraries monorepo (math libs)"
submodules = ["rocm-libraries"]
```

**What this means:** To build math libraries, git clone the `rocm-libraries` submodule.

#### Build Stage Definition

```toml
[build_stages.math-libs]
description = "Math libraries (BLAS, FFT, RAND, SOLVER, SPARSE)"
artifact_groups = ["math-libs"]
type = "per-arch"  # Built separately for each GPU architecture
```

**What this means:**
- Creates a CI job named "math-libs"
- Runs multiple times (once per GPU: gfx1150, gfx94x, gfx950, etc.)

#### Artifact Group Definition

```toml
[artifact_groups.math-libs]
description = "Math libraries (BLAS, FFT, RAND, etc.)"
type = "per-arch"
artifact_group_deps = ["hip-runtime", "profiler-core"]
source_sets = ["rocm-libraries", "rocm-systems", "math-libs"]
```

**What this means:**
- Contains multiple artifacts (blas, fft, rand, etc.)
- **Dependencies:** Must build hip-runtime and profiler-core first
- **Build type:** Separate builds for each GPU family

#### Artifact Definition

```toml
[artifacts.blas]
artifact_group = "math-libs"
type = "target-specific"  # Separate build per GPU
artifact_deps = ["core-hip", "rocprofiler-sdk"]
feature_group = "MATH_LIBS"
split_databases = ["rocblas", "hipblaslt"]
```

**What this means:**
- Artifact name: `blas`
- **Type:** `target-specific` = built once per GPU, kept separate
- **Dependencies:** Needs core-hip and rocprofiler-sdk built first
- **CMake feature:** Controlled by `THEROCK_ENABLE_MATH_LIBS`

### How Topology is Processed

#### At CMake Configure Time

```bash
cmake -B build -GNinja -DTHEROCK_AMDGPU_FAMILIES=gfx1150
```

**Processing flow:**

1. CMake runs `build_tools/topology_to_cmake.py`
2. Reads `BUILD_TOPOLOGY.toml`
3. Generates `build/cmake/topology_generated.cmake`

**Generated CMake variables:**

```cmake
# Artifact list
set(THEROCK_TOPOLOGY_ARTIFACTS
  "sysdeps" "base" "amd-llvm" "core-hip" "blas" "fft"
)

# Artifact types
set(THEROCK_ARTIFACT_TYPE_blas "target-specific")
set(THEROCK_ARTIFACT_TYPE_core-hip "target-neutral")

# Dependencies
set(THEROCK_ARTIFACT_DEPS_blas "core-hip;rocprofiler-sdk")

# CMake targets
add_custom_target(artifact-blas)
add_custom_target(artifact-group-math-libs)
add_dependencies(artifact-group-math-libs artifact-blas artifact-fft ...)
```

#### At CI Time

**CI script** `configure_ci.py` reads topology to generate build matrix:

```python
from _therock_utils.build_topology import BuildTopology

topology = BuildTopology.from_toml("BUILD_TOPOLOGY.toml")

for stage in topology.get_build_stages():
    if stage.type == "per-arch":
        # Create job for each GPU
        for gpu in ["gfx1150", "gfx94x", "gfx950"]:
            create_job(f"{stage.name}-{gpu}")
```

---

<a name="stage-1"></a>
## 3. Stage 1: Source Code to Portable Linux Artifacts

### 3.1 Starting Point: Source Code

**Repository structure:**
```
TheRock/
├── BUILD_TOPOLOGY.toml         ← Master configuration
├── core/
│   ├── clr/                    ← HIP runtime (git submodule)
│   │   ├── hipamd/src/hip_runtime.cpp
│   │   └── CMakeLists.txt
│   └── artifact-core-hip.toml  ← Component descriptor
├── math-libs/
│   └── BLAS/
│       ├── rocBLAS/            ← rocBLAS library (in submodule)
│       │   └── library/src/blas1/rocblas_axpy.cpp
│       └── artifact-blas.toml  ← Component descriptor
└── cmake/
    └── therock_artifacts.cmake ← Artifact build system
```

### 3.2 Build: Source → Stage Directories

**Command:**
```bash
cmake -B build -GNinja -DTHEROCK_AMDGPU_FAMILIES=gfx1150
ninja -C build
```

**Each component builds to its own stage directory:**

```
build/
├── core/clr/stage/
│   ├── bin/
│   │   ├── hipcc
│   │   ├── hipconfig
│   │   └── hipify-perl
│   ├── lib/
│   │   ├── libamdhip64.so.6.2.0         ← Actual shared library
│   │   ├── libamdhip64.so.6 → libamdhip64.so.6.2.0
│   │   ├── libamdhip64.so → libamdhip64.so.6
│   │   ├── libhiprtc.so.6.2.0
│   │   ├── libhiprtc.so.6 → libhiprtc.so.6.2.0
│   │   └── libhiprtc.so → libhiprtc.so.6
│   ├── include/
│   │   └── hip/
│   │       ├── hip_runtime.h
│   │       ├── hip_runtime_api.h
│   │       └── amd_detail/
│   │           └── hip_runtime_api.cpp
│   └── share/
│       └── hip/cmake/hip-config.cmake
│
└── math-libs/BLAS/rocBLAS/stage/
    ├── bin/
    │   ├── rocblas-bench               ← Benchmark executable
    │   ├── rocblas-test                ← Test executable
    │   └── rocblas/library/
    │       └── TensileLibrary_gfx1150.dat  ← GPU kernel data
    ├── lib/
    │   ├── librocblas.so.4.3.0         ← Actual library
    │   ├── librocblas.so.4 → librocblas.so.4.3.0
    │   ├── librocblas.so → librocblas.so.4
    │   └── rocblas/library/
    │       ├── TensileLibrary_gfx1150.co
    │       ├── TensileLibrary_gfx1150_Kernels.so
    │       └── TensileLibrary_fallback.dat
    ├── include/
    │   └── rocblas/
    │       ├── rocblas.h
    │       ├── rocblas-types.h
    │       ├── rocblas-functions.h
    │       └── internal/
    │           └── rocblas-device-functions.h
    └── share/
        ├── doc/rocblas/README.md
        └── rocblas/cmake/
            ├── rocblas-config.cmake
            └── rocblas-targets.cmake
```

### 3.3 Artifact Slicing: Stage → Component Artifacts

**Artifact descriptor defines component patterns:**

**File:** `core/artifact-core-hip.toml`
```toml
# HIP runtime artifact descriptor
[components.lib."core/clr/stage"]
include = [
  "lib/.hipInfo",
  "share/hip/**",
]

[components.run."core/clr/stage"]
include = [
  "bin/**",
]

[components.dev."core/clr/stage"]
# Uses defaults: include/**, share/**/cmake/**, **.a

[components.dbg."core/clr/stage"]
# Uses defaults: .build-id/**/*.debug
```

**File:** `math-libs/BLAS/artifact-blas.toml`
```toml
# rocBLAS component descriptor
[components.lib."math-libs/BLAS/rocBLAS/stage"]
include = [
  "bin/rocblas/library/**",      # Kernel data files
  "lib/rocblas/library/**",      # Compiled kernels
]

[components.dev."math-libs/BLAS/rocBLAS/stage"]
# Uses defaults: include/**, cmake/**

[components.test."math-libs/BLAS/rocBLAS/stage"]
include = [
  "bin/rocblas-bench*",
  "bin/rocblas-test*",
  "bin/rocblas_gentest.py",
]

[components.doc."math-libs/BLAS/rocBLAS/stage"]
# Uses defaults: share/doc/**
```

**Command:**
```bash
ninja -C build therock-archives
```

**Artifact builder processes each descriptor:**

```
build/artifacts/
├── core-hip_lib_generic/
│   ├── artifact_manifest.txt
│   ├── lib/
│   │   ├── libamdhip64.so.6.2.0
│   │   ├── libamdhip64.so.6 → libamdhip64.so.6.2.0
│   │   ├── libamdhip64.so → libamdhip64.so.6
│   │   ├── libhiprtc.so.6.2.0
│   │   ├── libhiprtc.so.6 → libhiprtc.so.6.2.0
│   │   └── libhiprtc.so → libhiprtc.so.6
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
│   ├── include/hip/
│   │   ├── hip_runtime.h
│   │   └── hip_runtime_api.h
│   └── share/hip/cmake/
│       └── hip-config.cmake
│
├── blas_lib_gfx1150/
│   ├── artifact_manifest.txt
│   ├── bin/rocblas/library/
│   │   └── TensileLibrary_gfx1150.dat
│   └── lib/
│       ├── librocblas.so.4.3.0
│       ├── librocblas.so.4 → librocblas.so.4.3.0
│       ├── librocblas.so → librocblas.so.4
│       └── rocblas/library/
│           ├── TensileLibrary_gfx1150.co
│           └── TensileLibrary_gfx1150_Kernels.so
│
├── blas_dev_gfx1150/
│   ├── artifact_manifest.txt
│   ├── include/rocblas/
│   │   ├── rocblas.h
│   │   └── rocblas-functions.h
│   └── share/rocblas/cmake/
│       └── rocblas-config.cmake
│
├── blas_test_gfx1150/
│   ├── artifact_manifest.txt
│   └── bin/
│       ├── rocblas-bench
│       ├── rocblas-test
│       └── rocblas_gentest.py
│
└── blas_doc_gfx1150/
    ├── artifact_manifest.txt
    └── share/doc/rocblas/
        └── README.md
```

**Compress into archives:**

```bash
# Creates .tar.xz files for each component
build/artifacts/
├── core-hip_lib_generic.tar.xz       (150 MB)
├── core-hip_run_generic.tar.xz       (5 MB)
├── core-hip_dev_generic.tar.xz       (20 MB)
├── blas_lib_gfx1150.tar.xz           (2.1 GB)
├── blas_dev_gfx1150.tar.xz           (500 KB)
├── blas_test_gfx1150.tar.xz          (50 MB)
└── blas_doc_gfx1150.tar.xz           (100 KB)
```

**Upload to S3:**

```
s3://therock-ci-artifacts/12345678-linux/
├── core-hip_lib_generic.tar.xz
├── core-hip_run_generic.tar.xz
├── core-hip_dev_generic.tar.xz
├── blas_lib_gfx1150.tar.xz
├── blas_dev_gfx1150.tar.xz
├── blas_test_gfx1150.tar.xz
└── blas_doc_gfx1150.tar.xz
```

---

<a name="stage-2"></a>
## 4. Stage 2: Portable Artifacts to Python Packages

### 4.1 Download Component Artifacts

**Command:**
```bash
python ./build_tools/fetch_artifacts.py \
  --run-id=12345678 \
  --artifact-group=gfx1150 \
  --output-dir=./artifacts
```

**Downloads from S3 and extracts:**
```
artifacts/
├── core-hip_lib_generic/
│   ├── lib/libamdhip64.so.6.2.0
│   └── lib/libhiprtc.so.6.2.0
├── core-hip_run_generic/
│   └── bin/hipcc
├── core-hip_dev_generic/
│   └── include/hip/hip_runtime.h
├── blas_lib_gfx1150/
│   └── lib/librocblas.so.4.3.0
└── blas_dev_gfx1150/
    └── include/rocblas/rocblas.h
```

### 4.2 Python Package Building

**Command:**
```bash
python ./build_tools/build_python_packages.py \
  --artifact-dir=./artifacts \
  --dest-dir=./packages \
  --version=7.12.0.dev0
```

#### Step 1: Build rocm-sdk-core Package

**Filter function:**

**File:** `build_tools/build_python_packages.py`
```python
def core_artifact_filter(an: ArtifactName) -> bool:
    core = an.name in [
        "core-hip",        # ← From BUILD_TOPOLOGY.toml
        "core-runtime",
        "base",
        "hipify",
        "rocprofiler-sdk",
        # ...
    ] and an.component in ["lib", "run"]

    # Special: HIP headers needed by hiprtc
    hip_dev = an.name in ["core-hip"] and an.component in ["dev"]

    return core or hip_dev
```

**Selected artifacts:**
- ✅ `core-hip_lib_generic` (name matches, component is lib)
- ✅ `core-hip_run_generic` (name matches, component is run)
- ✅ `core-hip_dev_generic` (special case for HIP headers)
- ❌ `blas_lib_gfx1150` (name "blas" not in core list)

**Package structure created:**
```
packages/rocm-sdk-core/
├── src/
│   └── rocm_sdk_core/
│       ├── __init__.py
│       ├── _dist_info.py
│       └── platform/
│           ├── bin/
│           │   └── hipcc
│           ├── lib/
│           │   ├── libamdhip64.so.6.2.0    ← SONAME only
│           │   └── libhiprtc.so.6.2.0      ← Symlinks removed
│           └── include/
│               └── hip/
│                   └── hip_runtime.h       ← From dev component
└── setup.py
```

**Key transformation:** Symlinks (`libamdhip64.so`, `libamdhip64.so.6`) removed because Python wheels don't support symlinks.

#### Step 2: Build rocm-sdk-libraries-gfx1150 Package

**Filter function:**
```python
def libraries_artifact_filter(target_family: str, an: ArtifactName) -> bool:
    return (
        an.name in ["blas", "fft", "rand", "rccl", "miopen"]  # ← From topology
        and an.component in ["lib"]
        and (an.target_family == target_family or an.target_family == "generic")
    )
```

**Selected artifacts:**
- ❌ `core-hip_lib_generic` (name not in libraries list)
- ✅ `blas_lib_gfx1150` (name="blas", component="lib", family matches)
- ❌ `blas_dev_gfx1150` (component="dev", not "lib")

**Package structure created:**
```
packages/rocm-sdk-libraries/
├── src/
│   └── rocm_sdk_libraries/
│       ├── __init__.py
│       ├── _dist_info.py
│       └── platform/
│           ├── bin/
│           │   └── rocblas/library/
│           │       └── TensileLibrary_gfx1150.dat
│           └── lib/
│               ├── librocblas.so.4.3.0    ← SONAME only
│               └── rocblas/library/
│                   └── TensileLibrary_gfx1150.co
└── setup.py
```

#### Step 3: Build rocm-sdk-devel Package

**Code:**
```python
devel = PopulatedDistPackage(params, logical_name="devel")
devel.populate_devel_files(
    addl_artifact_names=["prim", "rocwmma", "flatbuffers", "nlohmann-json"]
)
```

**What populate_devel_files() does:**
1. Includes all `dev` components from runtime artifacts
2. Creates **symlinks** to files in runtime packages (NO duplication!)
3. Stores everything in `_devel.tar.xz` (wheels can't contain symlinks)

**Package structure:**
```
packages/rocm-sdk-devel/
├── src/
│   └── rocm_sdk_devel/
│       ├── __init__.py
│       ├── _dist_info.py
│       └── _devel.tar.xz              ← Tarball inside wheel!
│           └── (contains when extracted:)
│               ├── include/
│               │   ├── rocblas/
│               │   │   └── rocblas.h
│               │   └── rocprim/       ← Header-only lib
│               │       └── rocprim.hpp
│               └── lib/
│                   ├── libamdhip64.so → ../../../_rocm_sdk_core_*/platform/lib/libamdhip64.so.6.2.0
│                   └── librocblas.so → ../../../_rocm_sdk_libraries_*/platform/lib/librocblas.so.4.3.0
└── setup.py
```

**CRITICAL:** Symlinks point to runtime package files using relative paths!

#### Step 4: Build rocm Selector Package

**Code:**
```python
PopulatedDistPackage(params, logical_name="meta")
```

**Package structure:**
```
packages/rocm/
├── src/
│   └── rocm_sdk/
│       ├── __init__.py
│       ├── _dist_info.py           ← GPU detection logic
│       ├── cli.py                  ← rocm-sdk command
│       └── test.py                 ← Self-tests
├── setup.py                        ← Dynamic dependency resolution
├── pyproject.toml
└── README.md
```

**No binary files** - pure Python only!

### 4.3 Build Wheels

```bash
# Internally called by build_python_packages.py
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

### 4.4 Upload to S3 PyPI Index

```
s3://therock-nightly-python/v2/gfx1150/
├── rocm-7.12.0.dev0.tar.gz
├── rocm_sdk_core-7.12.0.dev0-py3-none-linux_x86_64.whl
├── rocm_sdk_libraries_gfx1150-7.12.0.dev0-py3-none-linux_x86_64.whl
├── rocm_sdk_devel-7.12.0.dev0-py3-none-linux_x86_64.whl
└── index.html                        ← pip-compatible index
```

**Public URL:** `https://rocm.nightlies.amd.com/v2/gfx1150/`

---

<a name="stage-3"></a>
## 5. Stage 3: pip Installation and Dependency Resolution

### 5.1 User Command

```bash
python -m venv .venv
source .venv/bin/activate
pip install rocm[libraries,devel] --pre \
  --extra-index-url https://rocm.nightlies.amd.com/v2/gfx1150/
```

### 5.2 pip Processing Steps

#### Phase 1: Parse Package Specification

**Input:** `rocm[libraries,devel]`

**Parsed:**
- Package name: `rocm`
- Extras: `["libraries", "devel"]`
- Version: Latest pre-release allowed

#### Phase 2: Find Package

**pip searches:**
1. PyPI: `https://pypi.org/simple/rocm/` (not found)
2. Extra index: `https://rocm.nightlies.amd.com/v2/gfx1150/` (found!)

**Downloads:** `rocm-7.12.0.dev0.tar.gz`

#### Phase 3: GPU Detection During setup.py

**File:** `packages/rocm/setup.py`
```python
from pathlib import Path
from setuptools import setup

# Load _dist_info.py
dist_info_path = Path(__file__).parent / "src" / "rocm_sdk" / "_dist_info.py"
dist_info_globals = {}
exec(dist_info_path.read_text(), dist_info_globals)

# CRITICAL: Detect GPU at pip install time
determine_target_family = dist_info_globals["determine_target_family"]
detected_gpu = determine_target_family()  # Returns "gfx1150"

print(f"Detected GPU family: {detected_gpu}", file=sys.stderr)

# Build dependencies based on detected GPU
extras_require = {}

# Core always required
extras_require["core"] = ["rocm-sdk-core==7.12.0.dev0"]

# Libraries for detected GPU
extras_require["libraries"] = [
    f"rocm-sdk-libraries-{detected_gpu}==7.12.0.dev0"
    # Expands to: rocm-sdk-libraries-gfx1150==7.12.0.dev0
]

# Devel (GPU-agnostic)
extras_require["devel"] = ["rocm-sdk-devel==7.12.0.dev0"]

setup(
    name="rocm",
    version="7.12.0.dev0",
    install_requires=["rocm-sdk-core==7.12.0.dev0"],
    extras_require=extras_require,
    # ...
)
```

**GPU detection code:**

**File:** `src/rocm_sdk/_dist_info.py`
```python
def determine_target_family() -> str:
    """Detects GPU in priority order"""
    # 1. Environment variable override
    target = os.getenv("ROCM_SDK_TARGET_FAMILY")
    if target:
        return target

    # 2. Auto-detect using offload-arch tool
    target = discover_current_target_family()
    if target:
        return target

    # 3. Fallback to build-time default
    return DEFAULT_TARGET_FAMILY  # "gfx1150"

def discover_current_target_family() -> str | None:
    """Runs offload-arch to detect GPU"""
    result = subprocess.run(
        ["offload-arch", "--targets"],
        capture_output=True, text=True, timeout=5
    )
    if result.returncode == 0:
        # Output: "gfx1150\n"
        targets = result.stdout.strip().split("\n")
        return map_gfx_to_family(targets[0])
    return None
```

**On user's machine with gfx1150:**
```bash
$ offload-arch --targets
gfx1150
```
→ Returns `target_family = "gfx1150"`

#### Phase 4: Dependency Resolution

**From user request:** `rocm[libraries,devel]`

**Expands to:**
```
rocm==7.12.0.dev0
├── install_requires:
│   └── rocm-sdk-core==7.12.0.dev0
├── extras["libraries"]:
│   └── rocm-sdk-libraries-gfx1150==7.12.0.dev0   ← GPU detected!
└── extras["devel"]:
    └── rocm-sdk-devel==7.12.0.dev0
```

#### Phase 5: Download Packages

```
Downloading rocm_sdk_core-7.12.0.dev0-py3-none-linux_x86_64.whl (500 MB)
Downloading rocm_sdk_libraries_gfx1150-7.12.0.dev0-py3-none-linux_x86_64.whl (2.1 GB)
Downloading rocm_sdk_devel-7.12.0.dev0-py3-none-linux_x86_64.whl (300 MB)
```

#### Phase 6: Install to site-packages

```
.venv/lib/python3.12/site-packages/
├── rocm_sdk/
│   ├── __init__.py
│   └── cli.py
│
├── _rocm_sdk_core_7_12_0_dev0_123abc/
│   └── platform/
│       ├── bin/hipcc
│       └── lib/
│           ├── libamdhip64.so.6.2.0
│           └── libhiprtc.so.6.2.0
│
├── _rocm_sdk_libraries_gfx1150_7_12_0_dev0_456def/
│   └── platform/
│       └── lib/
│           └── librocblas.so.4.3.0
│
└── _rocm_sdk_devel_7_12_0_dev0_789ghi/
    └── _devel.tar.xz                 ← Extracted on first use
```

#### Phase 7: Extract Devel Tarball (on first access)

When user accesses devel files:

```python
from rocm_sdk import get_devel_path
devel_path = get_devel_path()  # Triggers extraction
```

**Extracted structure:**
```
.venv/lib/python3.12/site-packages/_rocm_sdk_devel_*/platform/
├── include/
│   ├── rocblas/rocblas.h
│   └── rocprim/rocprim.hpp
└── lib/
    ├── libamdhip64.so → ../../_rocm_sdk_core_*/platform/lib/libamdhip64.so.6.2.0
    └── librocblas.so → ../../_rocm_sdk_libraries_gfx1150_*/platform/lib/librocblas.so.4.3.0
```

**Symlinks now resolve correctly!**

---

<a name="library-duplication"></a>
## 6. Library Duplication: Storage Efficiency

### The Question

**Q:** If I install `rocm[libraries,devel]`, do I get `librocblas.so.4.3.0` stored twice?

**A:** **NO** - You get it **once** in the libraries package, and devel creates **symlinks** pointing to it.

### Proof

```bash
$ find .venv -name "librocblas.so*" -ls
```

**Output:**
```
2.1G  _rocm_sdk_libraries_gfx1150_*/platform/lib/librocblas.so.4.3.0    ← Actual file
   0  _rocm_sdk_devel_*/platform/lib/librocblas.so → ...                ← Symlink (0 bytes)
   0  _rocm_sdk_devel_*/platform/lib/librocblas.so.4 → ...              ← Symlink (0 bytes)
```

### Storage Breakdown

| Package | Contains | Storage |
|---------|----------|---------|
| `rocm-sdk-core` | HIP runtime binaries | 500 MB |
| `rocm-sdk-libraries-gfx1150` | Math library binaries | 2.1 GB |
| `rocm-sdk-devel` | Headers (300 MB) + symlinks (0 MB) | 300 MB |
| **Total** | | **2.9 GB** |

**NOT 5.1 GB** (which it would be if files were duplicated)

### How It Works

**Devel package creates relative symlinks:**

```
_rocm_sdk_devel_*/platform/lib/librocblas.so
    ↓ (symlink)
../../../_rocm_sdk_libraries_gfx1150_*/platform/lib/librocblas.so.4.3.0
```

The `{NONCE}` (e.g., `7_12_0_dev0_456def`) is filled in at package build time.

---

<a name="selector-mechanism"></a>
## 7. Advanced: How the rocm Selector Works

### The Problem

Different users have different GPUs:
- User A: gfx1150 (RX 7900 XTX)
- User B: gfx94x (MI300)
- User C: gfx950 (MI350)

But we want **one command**: `pip install rocm[libraries]`

### The Solution

**setup.py executes Python code at install time**, including GPU detection!

### Complete Flow

1. User runs: `pip install rocm[libraries]`
2. pip downloads: `rocm-7.12.0.dev0.tar.gz`
3. pip runs: `setup.py` to get metadata
4. **setup.py executes:**
   - Runs `offload-arch --targets` to detect GPU
   - Maps GPU to family (gfx1150 → gfx1150, gfx942 → gfx94x)
   - Dynamically builds `extras_require["libraries"]` with correct GPU
5. pip sees dependency: `rocm-sdk-libraries-gfx1150==7.12.0.dev0`
6. pip downloads and installs the GPU-specific wheel

### User Experience

**User with gfx1150:**
```bash
$ pip install rocm[libraries]
Collecting rocm
  Detected GPU family: gfx1150
Collecting rocm-sdk-libraries-gfx1150==7.12.0.dev0
  ...
```

**User with MI300 (gfx942):**
```bash
$ pip install rocm[libraries]
Collecting rocm
  Detected GPU family: gfx94x_dcgpu
Collecting rocm-sdk-libraries-gfx94x_dcgpu==7.12.0.dev0
  ...
```

**Same command, different packages installed!**

---

<a name="complete-example"></a>
## 8. Complete Example: rocBLAS End-to-End

This section traces **one library (rocBLAS)** through the entire pipeline.

### Stage 0: Topology Definition

**BUILD_TOPOLOGY.toml:**
```toml
[artifacts.blas]
artifact_group = "math-libs"
type = "target-specific"
artifact_deps = ["core-hip", "rocprofiler-sdk"]
feature_group = "MATH_LIBS"
```

### Stage 1: Source to Artifacts

**Source code:**
```
math-libs/rocm-libraries/rocBLAS/library/src/blas1/rocblas_axpy.cpp
```

**CMake builds:**
```
build/math-libs/BLAS/rocBLAS/stage/lib/librocblas.so.4.3.0
```

**Artifact descriptor slices:**
```toml
[components.lib."math-libs/BLAS/rocBLAS/stage"]
include = ["lib/rocblas/**"]
```

**Creates:**
```
build/artifacts/blas_lib_gfx1150.tar.xz (2.1 GB)
```

**Uploads to:**
```
s3://therock-ci-artifacts/12345678-linux/blas_lib_gfx1150.tar.xz
```

### Stage 2: Artifacts to Python

**Downloads:**
```
artifacts/blas_lib_gfx1150/lib/librocblas.so.4.3.0
```

**Python filter matches:**
```python
an.name == "blas"  # From topology
an.component == "lib"
an.target_family == "gfx1150"
```

**Creates wheel:**
```
rocm_sdk_libraries_gfx1150-7.12.0.dev0.whl
└── platform/lib/librocblas.so.4.3.0
```

**Uploads to:**
```
s3://therock-nightly-python/v2/gfx1150/rocm_sdk_libraries_gfx1150-*.whl
```

### Stage 3: pip Installation

**User runs:**
```bash
pip install rocm[libraries]
```

**setup.py detects GPU:**
```python
offload-arch --targets  # Returns: gfx1150
```

**pip installs:**
```
.venv/lib/python3.12/site-packages/
└── _rocm_sdk_libraries_gfx1150_*/platform/lib/librocblas.so.4.3.0
```

**User imports:**
```python
import torch
torch.cuda.is_available()  # Uses libamdhip64.so.6.2.0
torch.matmul(a, b)          # Uses librocblas.so.4.3.0
```

---

## Summary

### Key Points

1. **BUILD_TOPOLOGY.toml** is the single source of truth
   - Defines all artifacts, dependencies, and build structure
   - Drives CMake, CI, and packaging

2. **Component artifacts** are the reusable intermediate format
   - Created once in CI
   - Reused by Python, RPM, DEB packaging
   - Stored in S3 for pipeline use

3. **Python packaging** reorganizes components
   - Filters artifacts by name (from topology)
   - Removes symlinks (wheel limitation)
   - Creates GPU-specific packages

4. **pip installation** is dynamic
   - Detects GPU at install time
   - Resolves correct dependencies
   - No file duplication (symlinks in devel)

5. **Everything flows from topology**
   - Artifact names → Python filters
   - Dependencies → Build order
   - Types → GPU-specific builds

### Reference Documentation

- [BUILD_TOPOLOGY.toml](/BUILD_TOPOLOGY.toml) - Master configuration
- [Python Packaging Guide](/docs/packaging/python_packaging.md) - Usage guide
- [Versioning](/docs/packaging/versioning.md) - Version schemes
- [Artifact System](/cmake/therock_artifacts.cmake) - Artifact build system
