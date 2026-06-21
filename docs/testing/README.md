# Test Harness Documentation

Documentation for TheRock's generic, extensible, configuration-driven test harness.

---

## Quick Start

New to the test harness? Start here:

1. **[Test Harness Diagrams](test_harness_diagrams.md)** - Visual, presentation-ready diagrams
2. **[Quick Reference](quick_reference.md)** - Common tasks and patterns
3. **[Configuration-Driven Matrix](configuration_driven_matrix.md)** - How `fetch_test_configurations.py` works

For comprehensive details, see [Test Harness Architecture](test_harness_architecture.md).

---

## Documentation Overview

### 📊 [Test Harness Diagrams](test_harness_diagrams.md)

**Best for: Presentations, high-level understanding**

Easy-to-present visual diagrams showing:
- Three-layer architecture (test definition, orchestration, infrastructure)
- Label-based test selection
- Parallel execution via sharding
- Infrastructure independence
- End-to-end flow
- Key design benefits

Perfect for explaining the system to stakeholders or new team members.

---

### ⚙️ [Configuration-Driven Matrix](configuration_driven_matrix.md)

**Best for: Understanding how tests are orchestrated**

Deep dive into how `fetch_test_configurations.py` generates matrix jobs:
- How `test_matrix` dictionary defines all tests
- How `total_shards_dict` creates parallel jobs
- How environment variables filter configurations
- How GitHub Actions expands matrices
- Real examples with rocBLAS, MIOpen, rocRAND
- Visualizing configuration → matrix → jobs flow

**Emphasizes:**
- Configuration is data, not code
- Matrix expansion is automatic
- One config controls multiple dimensions
- No manual job dispatching needed

---

### 📖 [Test Harness Architecture](test_harness_architecture.md)

**Best for: Comprehensive reference, deep understanding**

Complete architecture documentation:
- Design principles and goals
- All architecture diagrams (detailed versions)
- Component responsibilities
- How to add new components (step-by-step)
- Special cases and advanced features

Use this when you need the full picture or are implementing something new.

---

### ⚡ [Quick Reference](quick_reference.md)

**Best for: Day-to-day development tasks**

Practical guide for common tasks:
- Adding tests to existing components
- Adding new components
- Label naming conventions
- Debugging tests
- Adjusting sharding
- Component-specific overrides
- Environment variables reference
- Common patterns
- Troubleshooting

Keep this open while working with tests!

---

## Key Concepts

### Generic & Extensible

- **One test runner** works for all components
- Components follow a **CTest labeling contract**
- Add components via **configuration only** (no code changes)

### Label-Based Configuration

- **Categories**: `quick`, `standard`, `comprehensive`, `full`
- **GPU architectures**: `ex_gpu_gfx1151`, `ex_gpu_gfx11X`
- **Exclusions**: `quick_exclude`, `standard_exclude`
- Tests selected automatically based on labels

### Configuration-Driven Matrix

- `test_matrix` dictionary defines all components
- `fetch_test_configurations.py` generates JSON matrix
- GitHub Actions expands matrix into parallel jobs
- **One config** → **Multiple components** → **Multiple shards per component** → **All running in parallel**

### Infrastructure Independence

- Runner pools managed separately by infra teams
- Test matrix specifies requirements, not specific runners
- GitHub Actions scheduler handles job allocation
- Weighted load balancing across runner pools

### Developer Experience

- Write tests in component repository
- Add CTest labels following convention
- Configure in `test_matrix` dictionary
- **No infrastructure knowledge required**
- **No manual job dispatching**

---

## Architecture at a Glance

```
Developer                  TheRock                   Infrastructure
   │                          │                            │
   │ Write tests              │                            │
   │ + labels                 │                            │
   ├─────────────────────────>│                            │
   │                          │                            │
   │                     Configure                         │
   │                     test_matrix                       │
   │                          │                            │
   │                   Generate matrix                     │
   │                          │                            │
   │                   ┌──────▼──────┐                    │
   │                   │GitHub Actions│                    │
   │                   │  Scheduler   │                    │
   │                   └──────┬──────┘                    │
   │                          │                            │
   │                          │  Match jobs to runners     │
   │                          ├───────────────────────────>│
   │                          │                            │
   │                          │                    Manage runners
   │                          │                    Add/remove pools
   │                          │                    Adjust capacity
   │<─────────────────────────┤                            │
   │  Test results            │                            │
```

**Key Points:**
- Each layer is **independent**
- Configuration **drives** job creation
- GitHub Actions **automates** job dispatch
- Infrastructure teams **scale** independently

---

## Common Workflows

### Adding a Test

1. Write test in component CMake
2. Add labels: `set_tests_properties(test PROPERTIES LABELS "standard;ex_gpu_gfx1100")`
3. Test locally: `ctest -L standard -L ex_gpu_gfx1100`
4. Push → Tests run automatically in CI

### Adding a Component

1. Add CTest labels to component tests
2. Add entry to `test_matrix` in `fetch_test_configurations.py`
3. Test locally with `test_runner.py`
4. Push → Component appears in test matrix

### Adjusting Parallelism

1. Edit `total_shards_dict` in `test_matrix`
2. Increase number = more parallel jobs (faster, needs more runners)
3. Decrease number = fewer parallel jobs (slower, fewer runners)
4. No other changes needed

### Scaling Infrastructure

1. Infra team provisions new runners
2. Add runners to pool with appropriate labels
3. Adjust weight in `amdgpu_family_matrix.py`
4. Jobs automatically distribute to new runners
5. No test configuration changes needed

---

## Documentation for Different Roles

### Test Authors (Component Developers)

Start with:
- [Quick Reference - Adding Tests](quick_reference.md#adding-tests-to-an-existing-component)
- [Quick Reference - Label Conventions](quick_reference.md#label-naming-conventions)
- [Diagrams - Label-Based Selection](test_harness_diagrams.md#2-label-based-test-selection-the-magic)

### Integration Engineers (TheRock Team)

Start with:
- [Quick Reference - Adding a Component](quick_reference.md#adding-a-new-component)
- [Configuration-Driven Matrix](configuration_driven_matrix.md)
- [Architecture - Component Responsibilities](test_harness_architecture.md#component-responsibilities)

### Infrastructure Teams

Start with:
- [Diagrams - Infrastructure Independence](test_harness_diagrams.md#4-infrastructure-independence)
- [Architecture - Infrastructure Layer](test_harness_architecture.md#architecture-diagrams)
- [Configuration-Driven Matrix - Filtering](configuration_driven_matrix.md#configuration-driven-filtering)

### Stakeholders / Presenters

Start with:
- [Test Harness Diagrams](test_harness_diagrams.md) - All diagrams are presentation-ready
- [Diagrams - Presentation One-Pager](test_harness_diagrams.md#7-presentation-one-pager)
- [Architecture - Design Principles](test_harness_architecture.md#design-principles)

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `build_tools/github_actions/fetch_test_configurations.py` | Generates test matrix from configuration |
| `build_tools/github_actions/test_executable_scripts/test_runner.py` | Generic test runner (label-based) |
| `.github/workflows/test_component.yml` | Reusable test workflow (matrix jobs) |
| `tests/amdgpu_family_matrix.py` | Runner pool configuration & weights |

---

## Design Highlights

### Why It's Powerful

1. **Separation of Concerns**
   - Test authors focus on tests
   - Infrastructure teams focus on hardware
   - GitHub handles everything in between

2. **Configuration Over Code**
   - `test_matrix` is pure data
   - No procedural job creation code
   - Easy to validate and modify

3. **Automatic Parallelization**
   - Configure `total_shards: 6`
   - Get 6 parallel jobs automatically
   - Linear speedup with more runners

4. **Zero Manual Dispatch**
   - No job assignment code
   - No runner selection logic
   - GitHub Actions matches jobs to runners

5. **Independent Scaling**
   - Add runners → More capacity
   - Adjust weights → Load balancing
   - No test changes needed

### What Makes It Extensible

- **Generic runner**: Works for any component with CTest labels
- **Label convention**: Simple contract, widely applicable
- **Configuration-driven**: Add component = add dict entry
- **Matrix expansion**: Automatic from configuration
- **Infrastructure abstraction**: Tests don't know about runners

---

## Getting Help

1. Check [Quick Reference](quick_reference.md) for common tasks
2. Review [Diagrams](test_harness_diagrams.md) for visual explanation
3. Read [Configuration-Driven Matrix](configuration_driven_matrix.md) for orchestration details
4. Consult [Architecture](test_harness_architecture.md) for comprehensive reference

For specific issues:
- Test not running? → [Quick Reference - Troubleshooting](quick_reference.md#troubleshooting)
- Adding component? → [Quick Reference - Adding New Component](quick_reference.md#adding-a-new-component)
- Understanding flow? → [Diagrams - End-to-End Flow](test_harness_diagrams.md#5-end-to-end-flow-one-picture)
- Configuration questions? → [Configuration-Driven Matrix](configuration_driven_matrix.md)

---

## Feedback

This documentation is maintained by the TheRock integration team. For corrections, improvements, or questions:
- Open an issue in the TheRock repository
- Submit a PR with documentation updates
- Contact the integration team

---

**Last Updated:** 2026-06-21
