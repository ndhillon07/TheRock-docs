# TheRock Development Guide

## Subfolders

- [Style guides](style_guides/)

## Pages

### Build system

- [Artifacts](artifacts.md)
- [Build System](build_system.md)
- [Dependencies](dependencies.md)
- [Development Guide](development_guide.md)
- [Installing Artifacts](installing_artifacts.md)
- [Sanitizers](sanitizers.md)
- **[Packaging Deep Dive](packaging_deep_dive.md)** - Complete guide covering all packaging types: portable artifacts → Python wheels, native Linux packages (DEB/RPM), and Windows packages

### Testing

- [Adding tests](adding_tests.md)
- [Test Debugging](test_debugging.md)
- [Test Filtering](test_filtering.md)
- [Test Environment Reproduction](test_environment_reproduction.md)
- [Test Runner Info](test_runner_info.md)
- [TheRock Test Harness](therock_test_harness.md)

### Infrastructure

- [Workflows Architecture](workflows_architecture.md)
- [Workflow Call Chains](workflow_call_chains.md)
- **[Exact CI Flow](EXACT_CI_FLOW.md)** - Deep dive: how configure_ci.py, BUILD_TOPOLOGY.toml, and amdgpu_family_matrix.py interact; proves generic builds are done once and reused
- [Release and Nightly Builds](release_and_nightly_builds.md)
- [GitHub Actions Debugging](github_actions_debugging.md)
- [CI Behavior Manipulation](ci_behavior_manipulation.md)

### Other topics

- [Git chores](git_chores.md)
- [Windows Support](windows_support.md)
