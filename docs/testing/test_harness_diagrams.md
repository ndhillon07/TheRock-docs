# Test Harness Architecture - Visual Diagrams

Easy-to-present diagrams explaining TheRock's generic, extensible test harness.

---

## 1. Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 1: TEST DEFINITION                      │
│                  (Component Developers Own This)                 │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Component Repository (e.g., rocBLAS, MIOpen)          │    │
│  │                                                         │    │
│  │  CMakeLists.txt:                                       │    │
│  │    add_test(NAME my_test ...)                          │    │
│  │    set_tests_properties(my_test PROPERTIES             │    │
│  │        LABELS "standard;ex_gpu_gfx1100")               │    │
│  │                                                         │    │
│  │  No infrastructure knowledge required!                 │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Developer writes tests ──> Adds labels ──> Done!              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Tests integrated into TheRock
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                 LAYER 2: TEST ORCHESTRATION                      │
│                (TheRock Integration Team Owns This)              │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  TheRock Configuration                                 │    │
│  │                                                         │    │
│  │  fetch_test_configurations.py:                         │    │
│  │    test_matrix = {                                     │    │
│  │        "rocblas": {                                     │    │
│  │            "timeout_minutes": 288,                     │    │
│  │            "total_shards": 6,                          │    │
│  │            "test_script": "test_runner.py"             │    │
│  │        }                                               │    │
│  │    }                                                   │    │
│  │                                                         │    │
│  │  Generic test_runner.py discovers & runs tests         │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Configure once ──> Works for all components                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Jobs dispatched via GitHub Actions
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   LAYER 3: INFRASTRUCTURE                        │
│                  (Infra Team Owns This)                          │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Runner Pools (Managed Independently)                  │    │
│  │                                                         │    │
│  │  ┌─────────────────┐    ┌─────────────────┐          │    │
│  │  │ gfx1100 Pool    │    │ gfx950 Pool     │          │    │
│  │  │ 12 runners      │    │ 6 runners       │          │    │
│  │  │ Labels: [....]  │    │ Labels: [....]  │          │    │
│  │  └─────────────────┘    └─────────────────┘          │    │
│  │                                                         │    │
│  │  Infra teams add/remove runners anytime                │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Scale up/down ──> No coordination with test authors needed    │
└─────────────────────────────────────────────────────────────────┘

KEY INSIGHT: Each layer is independent!
  ✓ Test authors never touch infrastructure
  ✓ Infrastructure teams never touch test definitions
  ✓ GitHub Actions automatically connects them
```

---

## 2. Label-Based Test Selection (The Magic!)

```
┌──────────────────────────────────────────────────────────────────┐
│                   COMPONENT TEST SUITE                            │
│               (Example: rocBLAS with 10,000 tests)               │
└──────────────────────────────────────────────────────────────────┘
                              │
            Every test has labels attached
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
   ┌─────────┐          ┌─────────┐          ┌─────────┐
   │Category │          │  GPU    │          │Exclude  │
   │Labels   │          │ Labels  │          │ Labels  │
   ├─────────┤          ├─────────┤          ├─────────┤
   │ quick   │          │gfx1100  │          │ quick_  │
   │standard │          │gfx110X  │          │ exclude │
   │compre-  │          │gfx11X   │          │standard_│
   │hensive  │          │gfx950   │          │ exclude │
   │ full    │          │gfx94X   │          │windows_ │
   └─────────┘          └─────────┘          │ exclude │
                                              └─────────┘

────────────────────────────────────────────────────────────────────

              TEST SELECTION AT RUNTIME

Input:
  ┌──────────────────────────────────────┐
  │ TEST_TYPE = "standard"               │
  │ AMDGPU_FAMILIES = "gfx1100"         │
  └──────────────────────────────────────┘

Generic test_runner.py logic:

  Step 1: Discover labels
    $ ctest --print-labels
    ──> [quick, standard, ex_gpu_gfx1100, ex_gpu_gfx110X, ...]

  Step 2: Match GPU architecture
    Current GPU: gfx1100
    Available:   [gfx1100, gfx110X, gfx11X]
    Best match:  gfx1100 (exact match wins!)

  Step 3: Build filter
    ctest -L standard -L ex_gpu_gfx1100

Result:
  ┌─────────────────────────────────────────────────────────┐
  │ ✓ Runs ~2,000 tests (standard + gfx1100 compatible)    │
  │ ✗ Skips 8,000 tests (wrong category or wrong GPU)      │
  └─────────────────────────────────────────────────────────┘

────────────────────────────────────────────────────────────────────

         WILDCARD MATCHING (Flexibility!)

Scenario: Component defines tests for GPU families, not every GPU

Component has labels:
  - ex_gpu_gfx115X  (covers gfx1150, gfx1151, gfx1152, gfx1153)
  - ex_gpu_gfx11X   (covers all gfx11XX GPUs)

Test runs on: gfx1151

Matching algorithm:
  1. Try exact match: gfx1151 ────> NOT FOUND
  2. Try gfx115X ─────────────────> FOUND! ✓
  3. (Would try gfx11X if above failed)

Result: Uses -L ex_gpu_gfx115X

This means:
  ✓ Component doesn't enumerate every GPU variant
  ✓ Works for future GPUs in the same family
  ✓ Test authors define families, not specific models
```

---

## 3. Parallel Execution via Sharding

```
┌──────────────────────────────────────────────────────────────────┐
│                    WITHOUT SHARDING                               │
│                  (Old approach - slow!)                           │
└──────────────────────────────────────────────────────────────────┘

Single Job:
  ┌────────────────────────────────────────────────────┐
  │  Run all 10,000 rocBLAS tests sequentially         │
  │  Time: 4 hours                                     │
  └────────────────────────────────────────────────────┘

Wall clock time: 4 hours ⏰

────────────────────────────────────────────────────────────────────

┌──────────────────────────────────────────────────────────────────┐
│                     WITH SHARDING                                 │
│              (Current approach - fast!)                           │
└──────────────────────────────────────────────────────────────────┘

Configuration:
  total_shards: 6

GitHub Actions creates 6 parallel jobs:

  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
  │ Shard 1/6        │  │ Shard 2/6        │  │ Shard 3/6        │
  │ ~1,666 tests     │  │ ~1,666 tests     │  │ ~1,666 tests     │
  │ Time: 40 min     │  │ Time: 40 min     │  │ Time: 40 min     │
  └──────────────────┘  └──────────────────┘  └──────────────────┘

  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
  │ Shard 4/6        │  │ Shard 5/6        │  │ Shard 6/6        │
  │ ~1,666 tests     │  │ ~1,666 tests     │  │ ~1,666 tests     │
  │ Time: 40 min     │  │ Time: 40 min     │  │ Time: 40 min     │
  └──────────────────┘  └──────────────────┘  └──────────────────┘

            All run in parallel on separate runners

Wall clock time: 40 minutes ⏰  (6x faster!)

────────────────────────────────────────────────────────────────────

              HOW SHARDING WORKS

CTest automatically distributes tests:

  ctest --tests-information 1,6  ──> Run tests 1, 7, 13, 19, ...
  ctest --tests-information 2,6  ──> Run tests 2, 8, 14, 20, ...
  ctest --tests-information 3,6  ──> Run tests 3, 9, 15, 21, ...
  ...

GTest automatically distributes tests:

  GTEST_SHARD_INDEX=0, GTEST_TOTAL_SHARDS=6
  GTEST_SHARD_INDEX=1, GTEST_TOTAL_SHARDS=6
  ...

Developers don't write sharding logic!
  ✓ Configure total_shards in test_matrix
  ✓ Test framework handles distribution
  ✓ Linear speedup with more shards
```

---

## 4. Infrastructure Independence

```
┌──────────────────────────────────────────────────────────────────┐
│          HOW INFRA TEAMS MANAGE RUNNERS INDEPENDENTLY             │
└──────────────────────────────────────────────────────────────────┘

Week 1: Initial Setup
──────────────────────

Infra team provisions hardware:

  gfx1100-pool-A: 10 runners
  gfx1100-pool-B: 5 runners

Configuration (in amdgpu_family_matrix.py):
  "test-runs-on-labels": [
      {"label": "gfx1100-pool-A", "weight": 70},
      {"label": "gfx1100-pool-B", "weight": 30}
  ]

Test jobs distribute automatically:
  70% → Pool A
  30% → Pool B

────────────────────────────────────────────────────────────────────

Week 5: Pool B Expansion
─────────────────────────

Infra team adds 5 more runners to Pool B:

  gfx1100-pool-A: 10 runners (unchanged)
  gfx1100-pool-B: 10 runners (5 added) ← New hardware

Updated configuration:
  "test-runs-on-labels": [
      {"label": "gfx1100-pool-A", "weight": 50},  ← Adjusted
      {"label": "gfx1100-pool-B", "weight": 50}   ← Adjusted
  ]

Test jobs now distribute:
  50% → Pool A
  50% → Pool B

Changes needed by test authors: ZERO ✓
Changes needed in test configs: ZERO ✓

────────────────────────────────────────────────────────────────────

Week 8: Pool A Maintenance
───────────────────────────

Infra team needs to refresh Pool A hardware:

Step 1: Drain Pool A
  Set weight to 0%:
    {"label": "gfx1100-pool-A", "weight": 0},
    {"label": "gfx1100-pool-B", "weight": 100}

  Result: All jobs → Pool B (temporarily)

Step 2: Update Pool A machines
  Replace hardware, update OS, etc.

Step 3: Restore Pool A
  Set weight back:
    {"label": "gfx1100-pool-A", "weight": 50},
    {"label": "gfx1100-pool-B", "weight": 50}

  Result: Jobs distribute across both pools again

During maintenance:
  ✓ Tests keep running (on Pool B)
  ✓ No test failures due to infrastructure work
  ✓ No developer awareness needed

────────────────────────────────────────────────────────────────────

Week 12: New GPU Architecture
──────────────────────────────

Infra team gets new gfx950 hardware:

Create new pool:
  gfx950-pool: 8 runners

Add to configuration:
  "gfx950": {
      "linux": {
          "test-runs-on-labels": [
              {"label": "gfx950-pool", "weight": 100}
          ]
      }
  }

Component developers:
  1. Add labels to tests: ex_gpu_gfx950
  2. That's it!

No job dispatch code, no runner management, no coordination needed!
```

---

## 5. End-to-End Flow (One Picture!)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          COMPLETE FLOW                                   │
│              From Code Push to Test Results                              │
└─────────────────────────────────────────────────────────────────────────┘

  Developer                  GitHub Actions              Infrastructure
     │                             │                            │
     │ git push                    │                            │
     ├────────────────────────────>│                            │
     │                             │                            │
     │                    ┌────────▼────────┐                  │
     │                    │ Workflow Trigger │                  │
     │                    └────────┬────────┘                  │
     │                             │                            │
     │                    ┌────────▼────────┐                  │
     │                    │fetch_test_      │                  │
     │                    │configurations.py│                  │
     │                    │                 │                  │
     │                    │Reads:           │                  │
     │                    │- test_matrix    │                  │
     │                    │- env vars       │                  │
     │                    │                 │                  │
     │                    │Generates:       │                  │
     │                    │- 50 job configs │                  │
     │                    │- shard arrays   │                  │
     │                    │- runner labels  │                  │
     │                    └────────┬────────┘                  │
     │                             │                            │
     │                    ┌────────▼────────┐                  │
     │                    │ Matrix Expansion│                  │
     │                    │ Creates 50 jobs │                  │
     │                    └────────┬────────┘                  │
     │                             │                            │
     │                        ─────┴─────                       │
     │                       ╱           ╲                      │
     │              ┌────────▼────┐ ┌───▼──────┐              │
     │              │Job: rocblas │ │Job: ...  │              │
     │              │shard 1/6    │ │          │              │
     │              │needs:       │ │needs:    │              │
     │              │gfx1100-pool │ │...       │              │
     │              └──────┬──────┘ └──────────┘              │
     │                     │                                    │
     │              GitHub Job Queue                            │
     │              Waiting for runner...                       │
     │                     │                                    │
     │                     │ Match job to runner based on labels
     │                     ├───────────────────────────────────>│
     │                     │                                    │
     │                     │                           ┌────────▼────────┐
     │                     │                           │ Runner Pool:    │
     │                     │                           │ gfx1100-pool-A  │
     │                     │                           │                 │
     │                     │                           │ Finds idle      │
     │                     │                           │ runner: #3      │
     │                     │                           └────────┬────────┘
     │                     │<─────────────────────────────────┤
     │                     │ Runner #3 assigned                 │
     │                     │                                    │
     │              ┌──────▼──────┐                            │
     │              │Execute Job: │                            │
     │              │             │                            │
     │              │1. Download  │                            │
     │              │   artifacts │                            │
     │              │2. Run test_ │                            │
     │              │   runner.py │                            │
     │              │3. Discover  │                            │
     │              │   labels    │                            │
     │              │4. Run ctest │                            │
     │              │5. Report    │                            │
     │              │   results   │                            │
     │              └──────┬──────┘                            │
     │                     │                                    │
     │<────────────────────┤                                    │
     │  Results (pass/fail)│                                    │
     │                     │                                    │
     │                     └───────────────────────────────────>│
     │                       Runner #3 becomes idle again       │

Developer sees:
  ✓ Test results
  ✗ No infrastructure details
  ✗ No job dispatching
  ✗ No runner management

Infrastructure team manages:
  ✓ Runner pools
  ✓ Hardware scaling
  ✗ No test definitions
  ✗ No job configurations
```

---

## 6. Key Design Benefits

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          SEPARATION OF CONCERNS                          │
└─────────────────────────────────────────────────────────────────────────┘

╔═══════════════════════════════════════════════════════════════════════╗
║  Test Authors Write:          Infrastructure Teams Manage:            ║
║  ───────────────────           ────────────────────────────           ║
║  • Test code                   • Runner hardware                      ║
║  • CTest labels                • Pool allocation                      ║
║  • Test categories             • Load balancing weights               ║
║                                • Capacity scaling                     ║
║  DON'T need to know:           DON'T need to know:                    ║
║  ✗ Which runners exist         ✗ What tests exist                     ║
║  ✗ Hardware capacity           ✗ Test categories                      ║
║  ✗ Load balancing              ✗ Sharding strategies                  ║
║  ✗ Job dispatching             ✗ Job configurations                   ║
╚═══════════════════════════════════════════════════════════════════════╝

────────────────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────┐
│                           EXTENSIBILITY                                  │
└─────────────────────────────────────────────────────────────────────────┘

Adding a new component requires:

  ┌────────────────────────────────────────────────────────────┐
  │ 1. Component: Add CTest labels (standard practice)         │
  └────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌────────────────────────────────────────────────────────────┐
  │ 2. TheRock: Add to test_matrix (5 lines of config)         │
  └────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌────────────────────────────────────────────────────────────┐
  │ 3. Done! Generic test_runner.py handles everything         │
  └────────────────────────────────────────────────────────────┘

NO changes needed:
  ✗ test_runner.py (generic for all components)
  ✗ test_component.yml (reusable workflow)
  ✗ Infrastructure (already has runners)
  ✗ Job dispatch logic (GitHub Actions handles it)

────────────────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────┐
│                          AUTOMATIC SCALING                               │
└─────────────────────────────────────────────────────────────────────────┘

Scenario: Test suite grows from 5,000 to 15,000 tests

  Before (5,000 tests):
    total_shards: 3
    Wall time: 45 min

  After (15,000 tests):
    total_shards: 6  ← Just change this number
    Wall time: 45 min (same!)

Linear scaling:
  ┌─────────────────────────────────────────────────────────┐
  │ Tests  │ Shards │ Wall Time │ Runner Count Needed      │
  ├────────┼────────┼───────────┼─────────────────────────┤
  │  5,000 │    3   │  45 min   │  3                       │
  │ 10,000 │    6   │  45 min   │  6                       │
  │ 15,000 │    9   │  45 min   │  9                       │
  │ 20,000 │   12   │  45 min   │ 12                       │
  └─────────────────────────────────────────────────────────┘

Infra team just adds runners. No test changes needed.

────────────────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────┐
│                        DEVELOPER EXPERIENCE                              │
└─────────────────────────────────────────────────────────────────────────┘

Traditional approach:
  1. Write test
  2. Figure out which runner to use
  3. Write job dispatch logic
  4. Configure job in workflow
  5. Wait for infra team to provision runner
  6. Debug job allocation issues
  7. Finally run test

TheRock approach:
  1. Write test
  2. Add label: "standard;ex_gpu_gfx1100"
  3. Done!

Time to first test run:
  Traditional: Days (waiting for infra + config)
  TheRock: Minutes (next PR push)

────────────────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────┐
│                      FAULT TOLERANCE                                     │
└─────────────────────────────────────────────────────────────────────────┘

What happens when a runner fails?

  Traditional (manual dispatch):
    ✗ Job hangs waiting for specific runner
    ✗ Manual intervention needed
    ✗ Tests blocked

  TheRock (automatic):
    ✓ GitHub Actions retries on another runner
    ✓ Job moves to different pool if needed
    ✓ Tests continue running
    ✓ No manual intervention

What happens when a runner pool is full?

  Traditional:
    ✗ Jobs queue up, waiting
    ✗ Long delays

  TheRock:
    ✓ Weighted load balancing spreads jobs
    ✓ Jobs use alternate pools
    ✓ Minimal queuing

What happens during maintenance?

  Traditional:
    ✗ Tests disabled during maintenance
    ✗ PR validation blocked

  TheRock:
    ✓ Set pool weight to 0%
    ✓ Jobs automatically use other pools
    ✓ Tests keep running
    ✓ Zero downtime
```

---

## 7. Presentation One-Pager

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│        TheRock Generic Test Harness: Architecture Overview              │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
│  Test Authors    │       │  Integration     │       │  Infrastructure  │
│                  │       │  (TheRock)       │       │  Teams           │
├──────────────────┤       ├──────────────────┤       ├──────────────────┤
│ Write tests      │──────>│ Configure once   │<──────│ Manage runners   │
│ Add labels       │       │ (test_matrix)    │       │ Add/remove pools │
│                  │       │                  │       │ Adjust weights   │
│ NO infrastructure│       │ Generic runner   │       │                  │
│    knowledge     │       │ (test_runner.py) │       │ NO test knowledge│
└──────────────────┘       └────────┬─────────┘       └──────────────────┘
                                    │
                                    │
                    ┌───────────────▼────────────────┐
                    │    GitHub Actions Scheduler    │
                    │    (Automatic Job Dispatch)    │
                    └───────────────┬────────────────┘
                                    │
                     ┌──────────────┼──────────────┐
                     │              │              │
                     ▼              ▼              ▼
              ┌──────────┐   ┌──────────┐   ┌──────────┐
              │ Pool A   │   │ Pool B   │   │ Pool C   │
              │ 12 runners│  │ 5 runners│   │ 8 runners│
              └──────────┘   └──────────┘   └──────────┘

═══════════════════════════════════════════════════════════════════════════

Key Principles:

  ✓ Label-Based Selection
    - Tests tagged with categories + GPU architectures
    - Generic runner discovers and filters automatically
    - No component-specific code

  ✓ Independent Scaling
    - Infra teams manage capacity without touching tests
    - Test authors add tests without knowing infrastructure
    - GitHub Actions connects them automatically

  ✓ Parallel Execution
    - Sharding for speed (configurable shards per component)
    - Linear scaling with more runners
    - 6x+ speedup typical

  ✓ Zero Manual Dispatch
    - Developers don't assign jobs to runners
    - Infrastructure doesn't configure which tests run where
    - Labels + GitHub Actions = automatic matching

═══════════════════════════════════════════════════════════════════════════

Example: Adding New Component

  Component Developer:             TheRock Integration:
  ───────────────────              ────────────────────
  1. Add CTest labels              1. Add 5-line config
     "standard;ex_gpu_gfx1100"        to test_matrix

  That's it! Generic runner handles the rest.

  Infrastructure: Nothing to do (runners already exist)

═══════════════════════════════════════════════════════════════════════════

Benefits:

  📈 Scalable:   Linear speedup with sharding
  🔌 Extensible: Add components via config only
  🎯 Focused:    Each team owns their domain
  ⚡ Fast:       Parallel execution, automatic dispatch
  🛡️  Robust:     Automatic retries, pool failover
  😊 Simple:     Write tests, add labels, done!
```

---

## Summary

The TheRock test harness achieves **separation of concerns** through:

1. **Labels** - Simple convention connects test definitions to execution
2. **Generic Runner** - One runner works for all components
3. **Matrix Jobs** - Automatic parallel execution via GitHub Actions
4. **Independent Pools** - Infrastructure scales without touching tests

**Result**: Developers write tests, infrastructure teams manage hardware, GitHub handles everything in between. No manual job dispatching, no coordination overhead, maximum flexibility.
