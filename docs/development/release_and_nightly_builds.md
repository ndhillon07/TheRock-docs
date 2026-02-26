# Release and Nightly Builds

This document explains how TheRock's release and nightly build workflows operate, including what commits they build from, where artifacts are stored, and how packages are versioned.

## Overview

TheRock has two separate scheduled workflows that produce artifacts:

| Workflow | Schedule | Purpose | Artifacts |
|----------|----------|---------|-----------|
| **CI Nightly** (`ci_nightly.yml`) | 2 AM UTC | Test all GPU families | Temporary CI artifacts for testing |
| **Release Nightly** (`release_portable_linux_packages.yml`) | 4 AM UTC | Produce user-facing packages | Permanent releases on CDN |

## CI Nightly vs Release Nightly

### CI Nightly (Testing)

> **Workflow:** [`.github/workflows/ci_nightly.yml`](../../.github/workflows/ci_nightly.yml)

**Purpose:** Validate that the codebase builds and tests pass across all GPU families.

**What it builds from:** HEAD of `main` branch at 2 AM UTC.

**Where artifacts go:**
- S3 bucket: `therock-ci-artifacts`
- Path: `s3://therock-ci-artifacts/{run_id}-linux/`
- Used by: Test workflows to download and run tests

**Artifact retention:** Temporary - used for CI testing, not for end users.

### Release Nightly (Distribution)

> **Workflow:** [`.github/workflows/release_portable_linux_packages.yml`](../../.github/workflows/release_portable_linux_packages.yml)

**Purpose:** Produce distributable packages that users can download and install.

**What it builds from:** HEAD of `main` branch at 4 AM UTC.

**Where artifacts go:**

| Artifact Type | S3 Bucket | Public URL |
|--------------|-----------|------------|
| Tarballs | `therock-nightly-tarball` | `https://rocm.nightlies.amd.com/tarball/` |
| Python wheels | `therock-nightly-python` | `https://rocm.nightlies.amd.com/v2/{gpu_family}/` |

**Note:** The release workflow runs 2 hours after CI nightly, but currently does **not** wait for CI to pass. There's a TODO in the codebase to add this validation.

## Package Versioning

Package versions are computed by [`build_tools/compute_rocm_package_version.py`](../../build_tools/compute_rocm_package_version.py) using the base version from [`version.json`](../../version.json) plus a suffix based on release type.

### Version Formats

| Release Type | Python Wheel Format | Example |
|-------------|---------------------|---------|
| **dev** | `{base}.dev0+{git_sha}` | `7.10.0.dev0+f689a8ea40232f3f` |
| **nightly** | `{base}a{YYYYMMDD}` | `7.10.0a20251021` |
| **prerelease** | `{base}rc{N}` | `7.10.0rc2` |

| Release Type | DEB Format | RPM Format |
|-------------|------------|------------|
| **dev** | `8.1.0~dev20251203` | `8.1.0~20251203gf689a8e` |
| **nightly** | `8.1.0~20251203` | `8.1.0~20251203` |
| **prerelease** | `8.1.0~pre2` | `8.1.0~rc2` |
| **release** | `8.1.0` | `8.1.0` |

### How Versions Are Unique

| Release Type | Unique By | Overwrites Previous? |
|-------------|-----------|---------------------|
| **dev** | Git commit SHA | No - each commit is unique |
| **nightly** | Date (YYYYMMDD) | No - each day accumulates |
| **prerelease** | Manual version number | No - each RC is unique |
| **release** | Base version only | Yes - same version overwrites |

## Artifact Naming

### Tarballs

Format: `therock-dist-{platform}-{gpu_family}-{version}.tar.gz`

Examples:
```
therock-dist-linux-gfx94X-dcgpu-7.10.0a20251021.tar.gz
therock-dist-linux-gfx94X-dcgpu-7.10.0a20251022.tar.gz
therock-dist-linux-gfx950-dcgpu-7.10.0a20251021.tar.gz
```

### Python Wheels

Format: `{package_name}-{version}-{python_tag}-{abi_tag}-{platform_tag}.whl`

Examples:
```
rocm_hip-7.10.0a20251021-py3-none-manylinux_2_28_x86_64.whl
rocm_hip-7.10.0a20251022-py3-none-manylinux_2_28_x86_64.whl
```

## S3 Bucket Structure

### Nightly Releases

```
s3://therock-nightly-tarball/
└── therock-dist-linux-gfx94X-dcgpu-7.10.0a20251021.tar.gz
└── therock-dist-linux-gfx94X-dcgpu-7.10.0a20251022.tar.gz
└── therock-dist-linux-gfx950-dcgpu-7.10.0a20251021.tar.gz
└── ...

s3://therock-nightly-python/
└── v2/
    └── gfx94X-dcgpu/
        ├── index.html                    # pip-compatible index
        ├── rocm_hip-7.10.0a20251021-*.whl
        ├── rocm_hip-7.10.0a20251022-*.whl
        └── ...
    └── gfx950-dcgpu/
        └── ...
```

### CI Artifacts (Temporary)

```
s3://therock-ci-artifacts/
└── {run_id}-linux/
    ├── index-gfx94X-dcgpu.html
    ├── therock-base-linux-gfx94X-dcgpu.tar.xz
    ├── therock-compiler-linux-gfx94X-dcgpu.tar.xz
    └── ...
└── {run_id}-windows/
    └── ...
```

## Release Types and Their Buckets

| Release Type | Tarball Bucket | Python Bucket | CloudFront URL |
|-------------|----------------|---------------|----------------|
| **nightly** | `therock-nightly-tarball` | `therock-nightly-python` | `rocm.nightlies.amd.com` |
| **dev** | `therock-dev-tarball` | `therock-dev-python` | `rocm.devreleases.amd.com` |
| **prerelease** | `therock-prerelease-tarball` | `therock-prerelease-python` | `rocm.prereleases.amd.com` |

## How to Install Nightly Packages

### Python Wheels

```bash
# Install latest nightly for a specific GPU family
pip install rocm-hip \
    --extra-index-url https://rocm.nightlies.amd.com/v2/gfx94X-dcgpu/

# Install a specific dated nightly
pip install rocm-hip==7.10.0a20251021 \
    --extra-index-url https://rocm.nightlies.amd.com/v2/gfx94X-dcgpu/
```

### Tarballs

```bash
# Download a specific nightly tarball
wget https://rocm.nightlies.amd.com/tarball/therock-dist-linux-gfx94X-dcgpu-7.10.0a20251021.tar.gz

# Extract
tar xzf therock-dist-linux-gfx94X-dcgpu-7.10.0a20251021.tar.gz
```

## Downstream Releases

When the release workflow completes, it triggers additional workflows to build:

| Downstream | Workflow | Triggered By |
|------------|----------|--------------|
| PyTorch wheels | `release_portable_linux_pytorch_wheels.yml` | Release workflow |
| JAX wheels | `release_portable_linux_jax_wheels.yml` | Release workflow |
| RPM packages | `build_native_linux_packages.yml` | Release workflow |
| DEB packages | `build_native_linux_packages.yml` | Release workflow |

## Manual Releases

For prereleases and official releases, use `workflow_dispatch` to trigger from a specific commit:

```yaml
# Inputs for manual release
inputs:
  release_type: "prerelease"  # or "dev", "nightly"
  prerelease_version: "2"     # for rc2
  ref: "v7.10.0-rc2"          # specific tag or commit
  families: "gfx94X,gfx950"   # which GPU families to build
```

This allows building from a **vetted, tested commit** rather than HEAD of main.

## Current Limitations

1. **No CI gating for nightly releases** - The release workflow at 4 AM does not wait for CI nightly (2 AM) to pass. There's a TODO to add this validation.

2. **Staging vs Production** - Artifacts are uploaded to both staging (`v2-staging/`) and production (`v2/`) directories. The intent is to test staging first, but promotion isn't gated.

## Related Documentation

- [Workflows Architecture](workflows_architecture.md) - Overall CI/CD workflow structure
- [CI Behavior Manipulation](ci_behavior_manipulation.md) - PR labels and CI controls
