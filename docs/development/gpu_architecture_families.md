# GPU Architecture Families

This document maps GPU chip IDs (like `gfx90a`) to their architecture families and explains the family naming convention.

## Quick Lookup: Find Your GPU's Family

> **Source:** [`cmake/therock_amdgpu_targets.cmake`](../../cmake/therock_amdgpu_targets.cmake)

| GPU Chip | Product Name | GPU Type | Architecture Families |
|----------|--------------|----------|----------------------|
| **gfx906** | Radeon VII / MI50 | Discrete | `dgpu-all`, `gfx906-dgpu` |
| **gfx908** | MI100 | Datacenter | `dcgpu-all`, `gfx908-dcgpu` |
| **gfx90a** | MI210/MI250 | Datacenter | `dcgpu-all`, `gfx90a-dcgpu` |
| **gfx942** | MI300A/MI300X | Datacenter | `dcgpu-all`, `gfx94X-all`, `gfx94X-dcgpu` |
| **gfx950** | MI350X/MI355X | Datacenter | `dcgpu-all`, `gfx950-all`, `gfx950-dcgpu` |
| **gfx1010** | AMD RX 5700 | Discrete | `dgpu-all`, `gfx101X-all`, `gfx101X-dgpu` |
| **gfx1011** | AMD Radeon Pro V520 | Discrete | `dgpu-all`, `gfx101X-all`, `gfx101X-dgpu` |
| **gfx1012** | AMD RX 5500 | Discrete | `dgpu-all`, `gfx101X-all`, `gfx101X-dgpu` |
| **gfx1030** | AMD RX 6800 / XT | Discrete | `dgpu-all`, `gfx103X-all`, `gfx103X-dgpu` |
| **gfx1031** | AMD RX 6700 / XT | Discrete | `dgpu-all`, `gfx103X-all`, `gfx103X-dgpu` |
| **gfx1032** | AMD RX 6600 | Discrete | `dgpu-all`, `gfx103X-all`, `gfx103X-dgpu` |
| **gfx1033** | AMD Van Gogh iGPU | Integrated | `igpu-all`, `gfx103X-all`, `gfx103X-igpu` |
| **gfx1034** | AMD RX 6500 XT | Discrete | `dgpu-all`, `gfx103X-all`, `gfx103X-dgpu` |
| **gfx1035** | AMD Radeon 680M Laptop | Integrated | `igpu-all`, `gfx103X-all`, `gfx103X-igpu` |
| **gfx1036** | AMD Raphael | Integrated | `igpu-all`, `gfx103X-all`, `gfx103X-igpu` |
| **gfx1100** | AMD RX 7900 XTX | Discrete | `dgpu-all`, `gfx110X-all`, `gfx110X-dgpu` |
| **gfx1101** | AMD RX 7800 XT | Discrete | `dgpu-all`, `gfx110X-all`, `gfx110X-dgpu` |
| **gfx1102** | AMD RX 7700S / Framework Laptop 16 | Discrete | `dgpu-all`, `gfx110X-all`, `gfx110X-dgpu` |
| **gfx1103** | AMD Radeon 780M Laptop | Integrated | `igpu-all`, `gfx110X-all`, `gfx110X-igpu` |
| **gfx1150** | AMD Strix Point | Integrated | `igpu-all`, `gfx115X-all`, `gfx115X-igpu` |
| **gfx1151** | AMD Strix Halo | Integrated | `igpu-all`, `gfx115X-all`, `gfx115X-igpu` |
| **gfx1152** | AMD Krackan 1 | Integrated | `igpu-all`, `gfx115X-all`, `gfx115X-igpu` |
| **gfx1153** | AMD Radeon 820M | Integrated | `igpu-all`, `gfx115X-all`, `gfx115X-igpu` |
| **gfx1200** | AMD RX 9060 / XT | Discrete | `dgpu-all`, `gfx120X-all` |
| **gfx1201** | AMD RX 9070 / XT | Discrete | `dgpu-all`, `gfx120X-all` |

## Family Naming Convention

GPU families use suffixes to indicate GPU categories:

| Suffix | Full Name | Description | Examples |
|--------|-----------|-------------|----------|
| `-dcgpu` | Datacenter GPU | Server GPUs with matrix cores, ECC memory | `gfx94X-dcgpu` (MI300X), `gfx950-dcgpu` (MI350X) |
| `-dgpu` | Discrete GPU | Desktop/workstation GPUs | `gfx110X-dgpu` (RX 7900), `gfx103X-dgpu` (RX 6800) |
| `-igpu` | Integrated GPU | Laptop/APU integrated graphics | `gfx110X-igpu` (Radeon 780M), `gfx115X-igpu` (Strix Point) |
| `-all` | All variants | Includes all GPU types in that architecture family | `gfx110X-all` (all RDNA3), `gfx120X-all` (all RDNA4) |

## How Family Membership Works

From `cmake/therock_amdgpu_targets.cmake`, GPUs are assigned to families using the `therock_add_amdgpu_target()` function:

```cmake
# Single GPU chip can belong to multiple families
therock_add_amdgpu_target(gfx942 "MI300A/MI300X CDNA"
    FAMILY dcgpu-all gfx94X-all gfx94X-dcgpu)
    # gfx942 belongs to 3 families:
    # - dcgpu-all: all datacenter GPUs (includes gfx908, gfx90a, gfx942, gfx950)
    # - gfx94X-all: all gfx94X variants (currently only gfx942)
    # - gfx94X-dcgpu: datacenter gfx94X GPUs (currently only gfx942)

# Multiple chips share families
therock_add_amdgpu_target(gfx1100 "AMD RX 7900 XTX"
    FAMILY dgpu-all gfx110X-all gfx110X-dgpu)
therock_add_amdgpu_target(gfx1103 "AMD Radeon 780M Laptop iGPU"
    FAMILY igpu-all gfx110X-all gfx110X-igpu)
    # Both gfx1100 and gfx1103 belong to gfx110X-all
    # But gfx1100 is dgpu, gfx1103 is igpu
```

## Family Groupings in CI

In CI workflows, GPUs are tested by family rather than individual chip IDs:

**Family Groups (from `new_amdgpu_family_matrix.py`):**

```python
# Presubmit (PRs)
["gfx94X-dcgpu", "gfx110X-all", "gfx1151", "gfx120X-all"]

# Postsubmit (main branch)
["gfx950-dcgpu"]

# Nightly
["gfx90X-dcgpu", "gfx101X-dgpu", "gfx103X-dgpu", "gfx1150", "gfx1152", "gfx1153"]
```

**What this means:**

| CI Entry | What Gets Tested | Specific Chips Included |
|----------|------------------|-------------------------|
| `gfx94X-dcgpu` | All datacenter gfx94X GPUs | gfx942 (MI300X) |
| `gfx90X-dcgpu` | CI grouping for older datacenter GPUs* | gfx906, gfx908, gfx90a |
| `gfx110X-all` | All RDNA3 GPUs (desktop + laptop) | gfx1100, gfx1101, gfx1102, gfx1103 |
| `gfx103X-dgpu` | Only discrete RDNA2 GPUs | gfx1030, gfx1031, gfx1032, gfx1034 (excludes gfx1033/1035/1036 iGPUs) |
| `gfx1151` | Only gfx1151 chip | gfx1151 (Strix Halo) |

**\*Important:** `gfx90X-dcgpu` is a **CI-level grouping alias**, not a cmake family. The individual chips have separate cmake families:
- gfx906 → `dgpu-all`, `gfx906-dgpu` (discrete GPU, not datacenter)
- gfx908 → `dcgpu-all`, `gfx908-dcgpu`
- gfx90a → `dcgpu-all`, `gfx90a-dcgpu`

CI groups them together as "gfx90X-dcgpu" for testing convenience, but each chip belongs to its own family in cmake.

**Why some GPUs are tested individually:**

- **gfx1150, gfx1152, gfx1153** are tested separately (not grouped as `gfx115X-igpu`) because each has different:
  - Build stability (gfx1150/1152/1153 expect failures, gfx1151 is stable)
  - Test runner availability (only gfx1151 has test runners)
  - CI configuration needs (different `expect_failure`, `run_tests` flags)

## Cross-Generation Families

Some families span multiple architecture generations:

| Family | Includes | Purpose |
|--------|----------|---------|
| `dcgpu-all` | gfx908, gfx90a, gfx942, gfx950 | All datacenter GPUs across generations |
| `dgpu-all` | gfx906, gfx1010-1012, gfx1030-1032/1034, gfx1100-1102, gfx1200-1201 | All discrete GPUs |
| `igpu-all` | gfx1033/1035/1036, gfx1103, gfx1150-1153 | All integrated GPUs |

## Finding Your GPU's Family

**Method 1: Check the source file**

Look up your chip ID in [`cmake/therock_amdgpu_targets.cmake`](../../cmake/therock_amdgpu_targets.cmake):

```bash
grep "gfx90a" cmake/therock_amdgpu_targets.cmake
# Output: therock_add_amdgpu_target(gfx90a "MI210/250 CDNA" FAMILY dcgpu-all gfx90a-dcgpu ...)
```

**Method 2: Use this table**

Refer to the [Quick Lookup](#quick-lookup-find-your-gpus-family) table at the top of this document.

**Method 3: Runtime detection**

On a system with your GPU:
```bash
rocminfo | grep "Name:"
# Or
offload-arch
# Output: gfx90a
```

Then look up the chip ID in this document.

## Related Documentation

- [Workflows Architecture](workflows_architecture.md) - How GPU families are used in CI
- [CI Behavior Manipulation](ci_behavior_manipulation.md) - Testing specific GPUs via labels
