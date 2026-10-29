## ADDED Requirements

### Requirement: Release and development version strings
The build SHALL have an option `NUVIO_RELEASE_BUILD` (default `OFF`). With it `ON`, the displayed version SHALL be exactly the CMake version (for example `0.1.0`). With it `OFF`, the displayed version SHALL be the CMake version followed by `-dev+<short git commit>`, or `-dev` when git is unavailable. The NACP version SHALL always be the plain CMake version.

#### Scenario: Development build
- **WHEN** a developer builds commit `a1b2c3d` without `NUVIO_RELEASE_BUILD`
- **THEN** the About screen shows "NuvioSwitch 0.1.0-dev+a1b2c3d"

#### Scenario: Release build
- **WHEN** the release workflow builds with `NUVIO_RELEASE_BUILD=ON`
- **THEN** the About screen shows "NuvioSwitch 0.1.0"

### Requirement: Pinned build container
The CI and release workflows SHALL use the same devkitPro container image, pinned to a specific tag and not `latest`, so a release can be rebuilt from its tag with the same toolchain.

#### Scenario: Reproducible rebuild
- **WHEN** the release workflow is re-run in `draft` mode on an existing tag's commit after the version is bumped
- **THEN** it uses the same container tag that is recorded in the workflow file
