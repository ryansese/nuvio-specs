## 1. Build system

- [x] 1.1 Add `.gitignore` for `build*/`, `*.nro`, `*.nacp`, `*.elf`, `.DS_Store`, `config.local.json`
- [x] 1.2 Add root `CMakeLists.txt`: borealis options and toolchain includes, `project(NuvioSwitch)`, version block, fail with a clear message if borealis isn't initialised, `add_subdirectory(library/borealis/library)`, `program_target`, link `borealis`, C++17
- [x] 1.3 Add `src/version.hpp.in` and generate `version.hpp` with `configure_file`
- [x] 1.4 Add the desktop branch: copy `resources/` next to the binary as a `.data` target
- [x] 1.5 Add the Switch branch: `switch_wrapper.c` from borealis, `NuvioSwitch.nro` target with `nacptool` and `elf2nro`, icon and romfs from `resources/`

## 2. Resources

- [x] 2.1 Copy borealis's required runtime resources (`font/`, `material/`, `i18n/en-US/hints.json`) into `resources/`
- [x] 2.2 Add `resources/i18n/en-US/app.json` with the app title, tab labels, placeholder notes and version text
- [x] 2.3 Add `resources/img/icon.jpg` (256×256, from NuvioTV's `app_logo_mark.png`) and the sidebar logo
- [x] 2.4 Copy NuvioTV's sidebar SVG icons (home, search, library, plugin, settings) into `resources/icon/`

## 3. App shell

- [x] 3.1 Add `src/core/app_info.{hpp,cpp}` with a pure function returning "NuvioSwitch <version>"
- [x] 3.2 Add `src/tabs/placeholder_tab.{hpp,cpp}` and `resources/xml/tabs/placeholder.xml` (centred title and note, `titleKey`/`noteKey` attributes; show the version on Settings)
- [x] 3.3 Add `resources/xml/activity/main.xml` (`AppletFrame` + `TabFrame` with the five tabs) and `src/activity/main_activity.{hpp,cpp}`
- [x] 3.4 Add `src/main.cpp`: init, window "NuvioSwitch", Nuvio dark theme colour overrides, register views, push `MainActivity`, main loop
- [x] 3.5 Check that + exits, B on tab content returns focus to the sidebar without exiting, and fix with an action on `MainActivity` if needed (borealis leaves global quit off by default, so `main.cpp` enables it; `TabFrame` already sends B on tab content to the sidebar. Confirmed on screen in 6.3/6.4)

## 4. Tests

- [x] 4.1 Add `tests/CMakeLists.txt` (desktop only) that fetches doctest and builds `NuvioSwitchTests` from `tests/*.cpp` and `src/core/*.cpp`
- [x] 4.2 Add `tests/app_info_test.cpp` covering the version string
- [x] 4.3 Register the tests with CTest and confirm `ctest --test-dir build_desktop` passes

## 5. CI

- [x] 5.1 Add `.github/workflows/build.yml` with a `switch` job (devkitpro/devkita64 container, `.nro` artifact) and a `desktop` job (macOS, build and `ctest`)
- [x] 5.2 Push and confirm both jobs pass (run 40911185784 on 875510e: both jobs green, .nro artifact uploaded)

## 6. Docs and verification

- [x] 6.1 Update NuvioSwitch's `CLAUDE.md` with the build, run, test and `nxlink` commands, and the layout of `src/`, `resources/` and `tests/`
- [x] 6.2 Add a short `README.md` (what it is, how to build, status)
- [ ] 6.3 Desktop check: build, run, switch through all tabs with keyboard or gamepad, see the theme and the version on Settings
- [ ] 6.4 Switch check: build the `.nro` and run it on hardware with `nxlink -s` or in an emulator; check tabs and exit with +
