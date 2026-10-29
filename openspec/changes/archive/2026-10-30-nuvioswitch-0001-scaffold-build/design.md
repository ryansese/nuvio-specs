## Context

`repos/NuvioSwitch` currently contains a `CLAUDE.md` and the `library/borealis` submodule (`xfangfang/borealis`, pinned at `5f08b28`). Borealis's own top-level `CMakeLists.txt` builds its demo app, and is the template for this one:

- `library/cmake/commonOption.cmake` defines the `PLATFORM_*` options and is included before `project()`.
- `library/cmake/toolchain.cmake` sets up the per-platform toolchain and provides `program_target()`, `gen_dksh()` and `add_libromfs()`.
- `add_subdirectory(library)` builds the `borealis` target.
- On Switch, a `<name>.nro` custom target runs `nacptool` and `elf2nro`, and copies `resources/` into romfs. It drops `resources/font`, because the system font is used on the console.

Borealis already bundles nlohmann/json (`library/include/borealis/extern/nlohmann`), tinyxml2, fmt, yoga and nanovg. NuvioTV is the reference for look and navigation only.

## Goals / Non-Goals

**Goals:**
- One source tree that builds and runs on macOS and builds a `.nro` for Switch.
- An app shell other changes can drop screens into: an `AppletFrame` containing a `TabFrame`, with one tab class per section.
- A unit-test target and CI, so later changes add tests without touching the build plumbing.

**Non-Goals:**
- Any networking, storage or real content; these belong to 0002 onwards.
- libmpv, libcurl or other Switch portlibs; they're added by the changes that need them.
- Windows/Linux desktop support, iOS, PS4 and PSV, even though borealis supports them.
- deko3d: the default OpenGL backend is used until playback (0008) shows a need for it.
- The Modern and Grid home layouts and the theme picker.

## Decisions

**1. Our own root `CMakeLists.txt`, copied and trimmed from borealis's demo, not borealis's file directly.**
- Set `BOREALIS_LIBRARY` to `library/borealis/library`, include `commonOption.cmake` and `toolchain.cmake` from there, then `add_subdirectory(library/borealis/library)`.
- Keep only the desktop (macOS) and Switch branches of the demo's per-platform blocks.
- Alternative: use borealis's root file through `add_subdirectory(library/borealis)`. Rejected, because it builds the demo and hard-codes its project name, icon and resources.

**2. Name, icon and version in one block.**
- `project(NuvioSwitch)`, version 0.1.0, author "Ryan Sese". The icon is `resources/img/icon.jpg`, made from NuvioTV's `app_logo_mark.png` and resized to the 256×256 JPEG that `elf2nro` expects.
- `configure_file` generates `version.hpp` from `src/version.hpp.in`, so the NACP and app code read the same version.
- The NACP gets no `--titleid`. An `.nro` doesn't need one, and the demo's `PROJECT_TITLEID` is never set anyway.

**3. Resources: borealis's required files plus ours.** The app's `resources/` contains:
- `i18n/en-US/app.json` with the app's strings, read in code as `"app/..."_i18n`;
- `xml/activity/main.xml` for the shell;
- `xml/tabs/*.xml` for the placeholders;
- `img/` for the icon and logo, `icon/` for the SVG sidebar icons from NuvioTV's `res/raw`;
- `material/MaterialIcons-Regular.ttf` and `font/switch_font.ttf` + `switch_icons.ttf`, copied from borealis's `resources/`. These are needed for the desktop build; the `.nro` step removes `font/`.
- `i18n/en-US/hints.json`, copied from borealis because its widgets reference those keys.

Not using libromfs. Plain files are simpler to edit, and they match what the demo does when `USE_LIBROMFS` is off.

**4. Code layout: `src/` holds code by layer, as in NuvioTV.**
- `src/main.cpp` covers init, theme overrides, view registration, the main activity and the loop.
- `src/activity/main_activity.{hpp,cpp}` and `src/tabs/placeholder_tab.{hpp,cpp}`. A single `PlaceholderTab` view takes `titleKey` and `noteKey` XML attributes, so five tabs don't need five classes; each later change replaces one with a real tab class.
- `src/core/` exists but stays empty except for `app_info.{hpp,cpp}`, which formats the "NuvioSwitch <version>" text. That function is pure, so it's the first thing tested.

**5. Theme overrides from code.**
- After `Application::init()`, call `Theme::getDarkTheme().addColor(...)` for `brls/background`, `brls/sidebar/background`, `brls/text`, `brls/text_disabled`, `brls/highlight/color1` and `color2` (both white), `brls/accent` and `brls/sidebar/active_item`.
- Then set `setThemeVariant(DARK)`.
- These are the borealis keys in `library/lib/core/theme.cpp`. Values come from NuvioTV's `PrimitiveTokens.kt`.

**6. Tests: doctest via FetchContent, desktop only.**
- `tests/CMakeLists.txt` is added only when `PLATFORM_DESKTOP` is on.
- It builds `NuvioSwitchTests` from `tests/*.cpp` plus `src/core/*.cpp`, without linking borealis, and calls `add_test`.
- doctest is a single header, header-only, and needs no extra install on CI.
- Alternative: Catch2. Rejected; it's heavier and v3 needs a compiled library.

**7. CI: two jobs in `.github/workflows/build.yml`.**
- `switch`: `container: devkitpro/devkita64`, checkout with `submodules: recursive`, then `cmake -B build_switch -DPLATFORM_SWITCH=ON` and `cmake --build build_switch --target NuvioSwitch.nro`. Uploads the `.nro`.
- `desktop`: `macos-latest`, checkout with `submodules: recursive`, then `cmake -B build_desktop -DPLATFORM_DESKTOP=ON`, build, and `ctest`.
- Borealis's macOS defaults build a universal binary (`arm64;x86_64`) with no extra dependencies. GLFW is vendored.

**8. Exit and B behaviour.** Keep borealis's global quit on + (don't call `setGlobalQuit(false)`). On a root tab, B moves focus back to the sidebar, which is `TabFrame`'s default; check that it doesn't pop the only activity.

## Risks / Trade-offs

- [Borealis's demo CMake assumes it is the root project (`CMAKE_SOURCE_DIR` paths for `switch_wrapper.c` and icons)] → Use `${BOREALIS_LIBRARY}`-based paths in our file, and check both builds in CI.
- [devkitPro container updates can break builds] → CI pins the image tag, picking a dated one if the latest breaks.
- [Fonts on desktop: borealis loads `font/switch_font.ttf` from resources] → Copy the borealis font files, as the demo does. Switch uses the system font, so it's unaffected.
- [The macOS universal build doubles compile time] → Acceptable for now; local builds can pass `-DCMAKE_OSX_ARCHITECTURES=arm64`.
- [B on a root tab might exit the app in this borealis version] → If so, register an action on the main activity that consumes B on root tabs and gives focus to the sidebar.

## Open Questions

None. The `.nro` icon question is settled: use NuvioTV's current logo mark (`app_logo_mark.png`), as Decision 2 describes.
