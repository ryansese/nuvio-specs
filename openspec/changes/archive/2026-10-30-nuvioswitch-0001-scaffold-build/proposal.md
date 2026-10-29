# NuvioSwitch-0001 Scaffold CMake build and app shell

## Why

NuvioSwitch is planned as a Switch homebrew port of NuvioTV, but it has no build and no code yet. Every later change in the roadmap (add-ons, catalogs, playback, sync) needs three things first: a project that builds for both macOS and the Switch from one source tree, an app window with the main navigation in place, and CI that catches a broken build.

## What Changes

- Add a root CMake project for NuvioSwitch that uses the bundled borealis library. It builds with `PLATFORM_DESKTOP` for macOS and `PLATFORM_SWITCH` for the console, and on Switch it packages a `NuvioSwitch.nro`.
- Set up the source layout: `src/` for app code, `resources/` for XML layouts, i18n, images and fonts, and `tests/` for unit tests.
- Add the app shell, a borealis tabbed frame with placeholder tabs Home, Search, Library, Add-ons and Settings, styled with NuvioTV's dark theme.
- Add a unit-test target (doctest) that builds on desktop only, with one smoke test.
- Add GitHub Actions CI that builds the Switch `.nro` in the devkitPro container and the macOS desktop build, and runs the unit tests.
- Update NuvioSwitch's `CLAUDE.md` with the real build, run and test commands.

## Capabilities

### New Capabilities
- `nuvioswitch-build`: how NuvioSwitch is built and packaged for desktop and Switch, the unit-test target, and CI.
- `nuvioswitch-app-shell`: the app window, the main tab navigation, the theme and what each tab shows before its feature exists.

### Modified Capabilities
None. `openspec/specs/` has no NuvioSwitch specs yet.

## Impact

- **Code:** new files in `repos/NuvioSwitch`: `CMakeLists.txt`, `src/`, `resources/`, `tests/`, `.github/workflows/`, `.gitignore`, and an updated `CLAUDE.md`. `library/borealis` is used as is and isn't edited.
- **Dependencies:**
  - borealis at its pinned commit `5f08b28`, including the nlohmann/json it bundles;
  - doctest, fetched for desktop test builds only;
  - the devkitPro toolchain (`devkitA64`, libnx, `switch-*` portlibs) for Switch builds.
- **Assets:** NuvioTV's DM Sans font, logo mark and sidebar SVG icons are copied into `resources/` for reuse. NuvioTV itself isn't changed.
- **Workspace:** the `repos/NuvioSwitch` pointer is bumped in nuvio-specs after the commit.
