## ADDED Requirements

### Requirement: Single CMake project for desktop and Switch
NuvioSwitch SHALL have one root `CMakeLists.txt` that builds the app from the same sources for macOS desktop (`-DPLATFORM_DESKTOP=ON`) and for Nintendo Switch (`-DPLATFORM_SWITCH=ON`), linking the borealis library from `library/borealis` without modifying it.

#### Scenario: Desktop build
- **WHEN** a developer on macOS runs `cmake -B build_desktop -DPLATFORM_DESKTOP=ON` and then `cmake --build build_desktop`
- **THEN** a runnable `NuvioSwitch` executable is produced, with the `resources/` folder copied next to it

#### Scenario: Switch build
- **WHEN** a developer with devkitPro installed runs `cmake -B build_switch -DPLATFORM_SWITCH=ON` and then `cmake --build build_switch --target NuvioSwitch.nro`
- **THEN** `build_switch/NuvioSwitch.nro` is produced with the app's icon, its NACP metadata (name "NuvioSwitch", author, version) and `resources/` packed as romfs

#### Scenario: Borealis not initialised
- **WHEN** CMake is configured and `library/borealis/library/CMakeLists.txt` does not exist
- **THEN** configuration fails with a message telling the developer to run `git submodule update --init --recursive`

### Requirement: Version defined in one place
The app version SHALL be defined once in the root `CMakeLists.txt` and used for the NACP version, the desktop window title and a generated `version.hpp` header available to app code.

#### Scenario: Version shown in the About placeholder
- **WHEN** the version in `CMakeLists.txt` is `0.1.0` and the app is built and run
- **THEN** the NACP reports `0.1.0` and the app's Settings placeholder shows "NuvioSwitch 0.1.0"

### Requirement: Unit-test target on desktop
The project SHALL provide a `NuvioSwitchTests` target, built only for desktop and registered with CTest, that compiles app logic sources without the UI and runs doctest test cases.

#### Scenario: Tests run with CTest
- **WHEN** a developer runs `ctest --test-dir build_desktop --output-on-failure` after a desktop build
- **THEN** the doctest suite runs and the command exits with status 0 when all tests pass

#### Scenario: Tests not built for Switch
- **WHEN** CMake is configured with `-DPLATFORM_SWITCH=ON`
- **THEN** no test target is created, and doctest is not downloaded

### Requirement: Continuous integration
The NuvioSwitch repository SHALL have a GitHub Actions workflow that runs on pushes and pull requests to `main`, builds the Switch `.nro` in the `devkitpro/devkita64` container, builds the desktop target on macOS, and runs the unit tests. The `.nro` SHALL be uploaded as a build artifact.

#### Scenario: Green build
- **WHEN** a commit that builds on both platforms and passes its tests is pushed to `main`
- **THEN** both CI jobs succeed and the run has a `NuvioSwitch.nro` artifact

#### Scenario: Broken build
- **WHEN** a commit fails to compile on either platform or a unit test fails
- **THEN** the corresponding CI job fails

### Requirement: Build outputs not committed
Build directories and local configuration SHALL be ignored by git.

#### Scenario: Clean status after build
- **WHEN** a developer builds for desktop and Switch in `build_desktop/` and `build_switch/`
- **THEN** `git status` in NuvioSwitch shows no untracked build files
