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

### How BUILD_TOPOLOGY.toml is Processed

This is the critical part: **BUILD_TOPOLOGY.toml is NOT read directly by CMake**. Instead:

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
│      - therock_add_feature() calls                          │
│      - CMake targets (artifact-blas, etc.)                  │
│      - Dependency variables                                 │
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

**Key insight:** BUILD_TOPOLOGY.toml is a data file, not a build script. Python reads it once at configure time and translates it to CMake code.

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

Let's understand each level with concrete examples.

### Level 1: Source Sets

**What they are:** Groupings of git submodules

**Why they exist:** TheRock has 20+ git submodules. CI jobs don't need all of them. Source sets let CI clone only what's needed.

**Example:**

```toml
[source_sets.compilers]
description = "Compiler toolchain submodules"
submodules = ["llvm-project", "HIPIFY", "spirv-llvm-translator"]

[source_sets.math-libs]
description = "Additional math library submodules"
submodules = ["libhipcxx"]

[source_sets.rocm-libraries]
description = "ROCm libraries monorepo (math libs)"
submodules = ["rocm-libraries"]  # This contains rocBLAS, rocFFT, etc.
```

**What happens:**

```bash
# Full checkout (gets all submodules)
./build_tools/fetch_sources.py

# Partial checkout (only compiler sources)
./build_tools/fetch_sources.py --stage compiler-runtime
```

**Fundamental concept:** Source sets are about **git repositories**, not build logic. They're just a convenience for partial checkouts.

### Level 2: Build Stages

**What they are:** Groups of components that build together in a single CI job

**Why they exist:** ROCm takes 4-8 hours to build everything. Build stages let CI parallelize by running multiple independent jobs.

**Example:**

```toml
[build_stages.foundation]
description = "Foundation - critical path dependencies"
artifact_groups = ["third-party-sysdeps", "base"]

[build_stages.compiler-runtime]
description = "Compiler, runtimes, and core profiling"
artifact_groups = [
    "compiler",
    "core-runtime",
    "hip-runtime",
    "profiler-core"
]

[build_stages.math-libs]
description = "Math and ML libraries per architecture"
artifact_groups = ["math-libs", "ml-libs"]
type = "per-arch"  # This means: run once per GPU family

[build_stages.comm-libs]
description = "Communication libraries per architecture"
artifact_groups = ["comm-libs"]
type = "per-arch"
```

**What happens in CI:**

```
┌───────────────┐
│ foundation    │  (Job 1: builds third-party deps and base)
└───────┬───────┘
        │
        ↓
┌────────────────────┐
│ compiler-runtime   │  (Job 2: builds compiler and HIP runtime)
└───────┬────────────┘
        │
        ├─────────────┬─────────────┐
        ↓             ↓             ↓
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ math-libs    │ │ math-libs    │ │ comm-libs    │
│ (gfx94X)     │ │ (gfx1100)    │ │ (gfx94X)     │
└──────────────┘ └──────────────┘ └──────────────┘
  Job 3           Job 4            Job 5
  (parallel)      (parallel)       (parallel)
```

**Fundamental concepts:**

1. **Build stage ≠ sequential step** - Many stages run in parallel
2. **type = "per-arch"** means CI creates one job per GPU family
3. **Stages define CI job boundaries**, not build system behavior

### Level 3: Artifact Groups

**What they are:** Logical groupings of related artifacts with shared dependencies

**Why they exist:** Components often have common dependencies and should be built together. Artifact groups express these relationships.

**Example:**

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

# Feature declaration (creates cache variable)
therock_add_feature(BLAS
  GROUP MATH_LIBS
  DESCRIPTION "Enables blas"
  REQUIRES HIP_RUNTIME  # Translated from artifact_deps
)

# This creates:
#   THEROCK_ENABLE_BLAS = ON/OFF  (user can toggle)

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
# 1. CMake configure - Python reads BUILD_TOPOLOGY.toml
cmake -B build -DTHEROCK_ENABLE_BLAS=ON -DTHEROCK_AMDGPU_FAMILIES=gfx94X-dcgpu

# Generated CMake code creates:
#   THEROCK_ENABLE_BLAS = ON
#   artifact-blas target
#   Dependencies: artifact-blas depends on artifact-core-hip

# 2. Build
ninja -C build artifact-blas

# CMake:
#   1. Checks if THEROCK_ENABLE_BLAS=ON (yes)
#   2. Builds dependencies first (core-hip)
#   3. Builds rocBLAS
#   4. Creates: build/artifacts/therock-blas-linux-gfx94X-dcgpu.tar.xz

# 3. CI uploads to S3
#   s3://therock-ci-artifacts/{run_id}-linux/therock-blas-linux-gfx94X-dcgpu.tar.xz
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

```
math-libs/BLAS/rocBLAS/stage/
├── bin/
│   ├── rocblas-bench
│   └── rocblas-test
├── lib/
│   ├── librocblas.so.4 → librocblas.so.4.0.0
│   ├── librocblas.so.4.0.0
│   └── rocblas/library/
│       └── TensileLibrary_gfx94X.dat
├── include/
│   └── rocblas/
│       └── rocblas.h
└── lib/cmake/
    └── rocblas/
        └── rocblas-config.cmake
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
