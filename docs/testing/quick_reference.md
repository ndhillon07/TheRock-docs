# Test Harness Quick Reference

Quick reference guide for common test harness tasks.

---

## Adding Tests to an Existing Component

### Step 1: Add test in component's CMakeLists.txt

```cmake
# Quick test (runs on all GPUs)
add_test(NAME my_quick_test COMMAND my_test --quick)
set_tests_properties(my_quick_test PROPERTIES
    LABELS "quick"
    TIMEOUT 300
)

# Standard test for specific GPU family
add_test(NAME my_standard_gfx1100 COMMAND my_test --standard)
set_tests_properties(my_standard_gfx1100 PROPERTIES
    LABELS "standard;ex_gpu_gfx110X"
    TIMEOUT 1800
)
```

### Step 2: Test locally

```bash
cd build/dist/rocm/bin/YourComponent
ctest -L quick --output-on-failure
```

### Step 3: Push and verify in CI

The test will automatically run in the appropriate test jobs.

---

## Adding a New Component

### 1. In test_runner.py (if directory name differs from job name)

```python
COMPONENT_DIR_MAPPING = {
    "your-component-job-name": "YourComponentDir",
}
```

### 2. In fetch_test_configurations.py

```python
test_matrix = {
    "your-component": {
        "job_name": "your-component",
        "fetch_artifact_args": "--your-component --tests",
        "timeout_minutes": 60,
        "test_script": f"python {_get_script_path('test_runner.py')}",
        "platform": ["linux", "windows"],
        "total_shards_dict": {
            "linux": 2,
            "windows": 1,
        },
    },
}
```

### 3. Test locally

```bash
export TEST_COMPONENT=your-component
export TEST_TYPE=quick
export AMDGPU_FAMILIES=gfx1100
export THEROCK_BIN_DIR=./build/dist/rocm/bin
export SHARD_INDEX=1
export TOTAL_SHARDS=1

python build_tools/github_actions/test_executable_scripts/test_runner.py
```

---

## Label Naming Conventions

### Category Labels

| Label | Purpose | Typical Time |
|-------|---------|--------------|
| `quick` | Smoke tests for rapid feedback | 5-15 min |
| `standard` | Pre-commit validation | 30-60 min |
| `comprehensive` | Extended coverage | 2-4 hours |
| `full` | Complete test suite | 4+ hours |

### GPU Exclusion Labels

Format: `ex_gpu_{architecture}`

**Exact matches:**
- `ex_gpu_gfx1151` - Only for gfx1151
- `ex_gpu_gfx950` - Only for gfx950

**Wildcard families:**
- `ex_gpu_gfx115X` - Matches gfx1150, gfx1151, gfx1152, gfx1153
- `ex_gpu_gfx11X` - Matches all gfx11XX GPUs
- `ex_gpu_gfx94X` - Matches gfx940, gfx941, gfx942

**Matching priority:** Exact > Most specific wildcard > Less specific wildcard

Example:
```cmake
# This test should ONLY run on gfx115X family GPUs
set_tests_properties(my_test PROPERTIES
    LABELS "standard;ex_gpu_gfx115X"
)
```

### Exclude Labels

| Label | Purpose |
|-------|---------|
| `quick_exclude` | Skip this test in quick runs |
| `standard_exclude` | Skip this test in standard runs |
| `windows_exclude` | Skip on Windows (handled by platform filter) |

---

## Common Test Configurations

### Fast smoke test (all GPUs)

```cmake
add_test(NAME smoke_test COMMAND my_test --smoke)
set_tests_properties(smoke_test PROPERTIES
    LABELS "quick"
    TIMEOUT 300
)
```

### GPU-specific comprehensive test

```cmake
add_test(NAME comprehensive_gfx1100 COMMAND my_test --comprehensive)
set_tests_properties(comprehensive_gfx1100 PROPERTIES
    LABELS "comprehensive;ex_gpu_gfx1100"
    TIMEOUT 7200
)
```

### Test that should be excluded from quick runs

```cmake
add_test(NAME slow_integration_test COMMAND my_test --integration)
set_tests_properties(slow_integration_test PROPERTIES
    LABELS "standard;quick_exclude"
    TIMEOUT 3600
)
```

### Multi-label test (runs in multiple categories)

```cmake
add_test(NAME important_test COMMAND my_test --important)
set_tests_properties(important_test PROPERTIES
    LABELS "quick;standard;comprehensive"
    TIMEOUT 600
)
```

---

## Debugging Tests

### Check which tests would run

```bash
export TEST_COMPONENT=rocblas
export TEST_TYPE=standard
export AMDGPU_FAMILIES=gfx1100
export THEROCK_BIN_DIR=./build/dist/rocm/bin

cd build/dist/rocm/bin/rocblas
ctest -L standard -L ex_gpu_gfx1100 -N
```

### Check available labels

```bash
cd build/dist/rocm/bin/rocblas
ctest --print-labels
```

### Run specific test

```bash
cd build/dist/rocm/bin/rocblas
ctest -R my_test_name --output-on-failure -V
```

### Run tests matching pattern

```bash
cd build/dist/rocm/bin/rocblas
ctest -R "gemm.*gfx1100" --output-on-failure
```

### Dry run (show what would execute)

```bash
export DRY_RUN=1
python build_tools/github_actions/test_executable_scripts/test_runner.py
```

---

## Adjusting Sharding

### Increase parallelism (faster, needs more runners)

In `fetch_test_configurations.py`:

```python
"total_shards_dict": {
    "linux": 12,  # Was 6, now 12 (2x more parallel jobs)
    "windows": 4,
},
```

### Decrease parallelism (slower, fewer runners needed)

```python
"total_shards_dict": {
    "linux": 2,  # Was 6, now 2 (fewer parallel jobs)
    "windows": 1,
},
```

### Calculate optimal shards

Target: ~30-45 min per shard

```
Optimal shards = Total test time / Target time per shard

Example:
  Total time (sequential): 3 hours = 180 min
  Target per shard: 30 min
  Optimal shards: 180 / 30 = 6 shards
```

---

## Component-Specific Overrides

### Custom test directory

```python
COMPONENT_OVERRIDES = {
    "my-component": {
        "test_dir": ["custom", "path", "to", "tests"],
    }
}
```

### Test directory based on TEST_TYPE

```python
COMPONENT_OVERRIDES = {
    "my-component": {
        "test_dir_by_type": {
            "quick": ["bin", "quick_tests"],
            "standard": ["bin", "standard_tests"],
        },
    }
}
```

### Additional environment paths

```python
COMPONENT_OVERRIDES = {
    "my-component": {
        "additional_env_paths": {
            "PATH": [["bin"], ["bin", "tools"]],
            "LD_LIBRARY_PATH": [["lib"], ["lib", "extra"]],
        },
    }
}
```

### Build tree paths (for libraries not in install tree)

```python
COMPONENT_OVERRIDES = {
    "my-component": {
        "env_prepend_from_therock": {
            "LD_LIBRARY_PATH": [
                ["build", "my-component", "dist", "lib"],
            ],
        },
    }
}
```

---

## Test Matrix Configuration Options

### Basic configuration

```python
"my-component": {
    "job_name": "my-component",
    "fetch_artifact_args": "--my-component --tests",
    "timeout_minutes": 60,
    "test_script": f"python {_get_script_path('test_runner.py')}",
    "platform": ["linux", "windows"],
    "total_shards_dict": {"linux": 2, "windows": 1},
}
```

### With custom container

```python
"my-component": {
    # ... basic config ...
    "container_image": "ghcr.io/rocm/custom_image@sha256:abc123...",
}
```

### With additional container options

```python
"my-component": {
    # ... basic config ...
    "container_options": ["--cap-add=SYS_PTRACE", "--privileged"],
}
```

### With additional Python requirements

```python
"my-component": {
    # ... basic config ...
    "additional_requirements_files": [
        "share/my-component/requirements.txt",
    ],
}
```

### Excluding specific GPU families

```python
"my-component": {
    # ... basic config ...
    "exclude_family": {
        "linux": ["gfx1150", "gfx1151"],
        "windows": ["gfx1150"],
    },
}
```

### Multi-GPU tests

```python
"my-component": {
    # ... basic config ...
    "multi_gpu": {
        "linux": ["gfx94X-dcgpu", "gfx950-dcgpu"],
    },
}
```

### Expected failures (informational runs)

```python
"my-component": {
    # ... basic config ...
    "expect_failure": True,  # Job won't fail the workflow
}
```

---

## Environment Variables Reference

### At Configuration Time (fetch_test_configurations.py)

| Variable | Purpose | Example |
|----------|---------|---------|
| `PROJECTS_TO_TEST` | Which components to test | `"rocblas,miopen"` or `"*"` |
| `AMDGPU_FAMILIES` | Target GPU architecture | `"gfx1100"` |
| `TEST_TYPE` | Test category | `"quick"`, `"standard"`, `"comprehensive"`, `"full"` |
| `TEST_LABELS` | Filter by specific labels | `["test:rocblas", "test:miopen"]` |
| `RUN_EXTENDED_TESTS` | Include benchmark/functional | `"true"` or `"false"` |

### At Test Execution Time (test_runner.py)

| Variable | Purpose | Example |
|----------|---------|---------|
| `TEST_COMPONENT` | Component being tested | `"rocblas"` |
| `TEST_TYPE` | Test category to run | `"standard"` |
| `AMDGPU_FAMILIES` | Current GPU architecture | `"gfx1100"` |
| `SHARD_INDEX` | Current shard (1-indexed) | `"2"` |
| `TOTAL_SHARDS` | Total number of shards | `"6"` |
| `THEROCK_BIN_DIR` | Binary directory | `"./build/dist/rocm/bin"` |
| `ROCM_PATH` | ROCm installation root | `"./build/dist/rocm"` |

### GTest Sharding (auto-set by test_runner.py)

| Variable | Purpose |
|----------|---------|
| `GTEST_SHARD_INDEX` | GTest shard index (0-indexed) |
| `GTEST_TOTAL_SHARDS` | Total GTest shards |

---

## Common Patterns

### Pattern: Tests for GPU family with fallback

```cmake
# Preferred: Specific GPU test
add_test(NAME test_gfx1151 COMMAND my_test --arch=gfx1151)
set_tests_properties(test_gfx1151 PROPERTIES
    LABELS "standard;ex_gpu_gfx1151"
)

# Fallback: Family test
add_test(NAME test_gfx115X COMMAND my_test --arch=gfx115X)
set_tests_properties(test_gfx115X PROPERTIES
    LABELS "standard;ex_gpu_gfx115X"
)
```

### Pattern: Same test, multiple categories

```cmake
add_test(NAME smoke_test COMMAND my_test --quick)
set_tests_properties(smoke_test PROPERTIES
    LABELS "quick;standard;comprehensive;full"
)
```

### Pattern: Progressive test coverage

```cmake
# Quick: Just smoke tests
add_test(NAME quick_smoke COMMAND my_test --smoke)
set_tests_properties(quick_smoke PROPERTIES LABELS "quick")

# Standard: Smoke + basic validation
add_test(NAME standard_basic COMMAND my_test --basic)
set_tests_properties(standard_basic PROPERTIES LABELS "standard")

# Comprehensive: Standard + extended
add_test(NAME comprehensive_extended COMMAND my_test --extended)
set_tests_properties(comprehensive_extended PROPERTIES LABELS "comprehensive")

# Full: Everything
add_test(NAME full_all COMMAND my_test --all)
set_tests_properties(full_all PROPERTIES LABELS "full")
```

### Pattern: Platform-specific tests

```cmake
if(WIN32)
    add_test(NAME windows_specific COMMAND my_test --windows)
    set_tests_properties(windows_specific PROPERTIES LABELS "standard")
else()
    add_test(NAME linux_specific COMMAND my_test --linux)
    set_tests_properties(linux_specific PROPERTIES LABELS "standard")
endif()
```

---

## Troubleshooting

### Issue: Tests not running in CI

**Check:**
1. Component in `PROJECTS_TO_TEST`? (or using `*`)
2. Platform matches? (`"platform": ["linux", "windows"]`)
3. GPU not excluded? (`exclude_family` config)
4. Labels correct? (`ctest --print-labels`)

### Issue: Wrong tests running

**Check:**
1. Label prefix correct? (`ex_gpu_` not `exclude_gpu_`)
2. GPU matching working? (check test_runner.py logs)
3. Category label present? (`quick`, `standard`, etc.)

### Issue: Tests timing out

**Solutions:**
1. Increase `timeout_minutes` in test_matrix
2. Add `TIMEOUT` property to CTest test
3. Increase sharding (split into more parallel jobs)

### Issue: Tests not sharding correctly

**Check:**
1. GTest-based? Ensure `GTEST_SHARD_INDEX` is set
2. CTest-based? Uses `--tests-information` flag
3. Verify `total_shards` > 1 in config

### Issue: All shards running same tests

**Cause:** Test framework doesn't support sharding

**Solutions:**
1. For GTest: Ensure env vars `GTEST_SHARD_INDEX`, `GTEST_TOTAL_SHARDS` are read
2. For CTest: Already handled by `--tests-information`
3. For custom: Implement sharding in test script

---

## Best Practices

### ✓ DO

- Use wildcard labels for GPU families (`ex_gpu_gfx11X`)
- Keep quick tests fast (< 15 min total)
- Use appropriate timeouts per test
- Test locally before pushing
- Use meaningful test names
- Document complex test configurations

### ✗ DON'T

- Enumerate every GPU variant (use wildcards)
- Make quick tests comprehensive (defeats the purpose)
- Set timeouts too low (causes flaky failures)
- Skip local testing (catches issues early)
- Use component-specific code in test_runner.py (keep it generic)
- Hardcode runner names in test configs (use labels)

---

## Getting Help

1. **Read the logs**: Test runner prints discovery and matching steps
2. **Test locally**: Faster iteration than CI
3. **Check examples**: Look at existing components (rocBLAS, MIOpen)
4. **Verify labels**: `ctest --print-labels` shows what's available
5. **Ask questions**: Include test_runner.py output in questions
