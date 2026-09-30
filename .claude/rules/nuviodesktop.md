---
paths:
  - "repos/NuvioDesktop/**"
---

# NuvioDesktop (repos/NuvioDesktop)

This file provides guidance to Claude Code (claude.ai/code) when working with code in the NuvioDesktop submodule.

## What this is

Nuvio Desktop: a Kotlin Multiplatform / Compose Multiplatform media client for Windows, macOS and Linux (alpha). The repo is a single Gradle build (root project name `Nuvio`) that also carries the Android and iOS targets of the shared `composeApp` module. Desktop is the focus; `androidApp/`, `iosApp/`, `MPVKit/` (the only real git submodule, see `.gitmodules`) and `libass-android/` exist for the mobile targets.

## Commands

Use the repo's `./gradlew` (`.\gradlew.bat` on Windows).

```bash
./gradlew :composeApp:run                                   # run desktop app from source
./gradlew :composeApp:desktopTest                           # desktop unit/UI tests
./gradlew :composeApp:desktopTest --tests "com.nuvio.app.features.home.HomeHeroPagerTest"   # single test class
./gradlew :composeApp:packageReleaseDistributionForCurrentOS # release package for this host
./gradlew :composeApp:packageReleaseMsi --rerun-tasks        # Windows
./scripts/build-macos-release-dmgs.sh --package-only         # macOS DMGs (per-arch)
./gradlew :composeApp:packageReleaseDeb                      # Linux (also Rpm, AppImage)
```

Common tests live in `commonTest` (run via the desktop target); there is no lint task. CI is release-oriented (`.github/workflows/desktop-release.yml`, modes `build-only|dry-run|draft|publish`).

Version bumps go through `./scripts/set-version.sh --desktop <name> --desktop-code <n>` (`--show` to print). The source of truth is `composeApp/Configuration/DesktopVersion.properties`; `release-metadata.sh` derives release info from its git history.

Android builds require choosing a distribution: `-Pnuvio.android.distribution=full|playstore` (also `NUVIO_ANDROID_DISTRIBUTION` env or `local.properties`); iOS uses `nuvio.ios.distribution`.

## Architecture

**Source sets** (`composeApp/src/`): `commonMain` holds nearly everything (UI, features, repositories). `desktopMain` has JVM desktop actuals and integrations; `nonWindowsDesktopMain` / `windowsDesktopMain` split OS-specific desktop code; `androidMain`/`iosMain` and the distribution sets (`androidFull`, `androidPlaystore`, `fullCommonMain`, `iosFull`, `iosAppStore`) are the other targets. Shared code uses `expect`/`actual` (e.g. `Platform.kt` with `Platform.desktop.kt`), so a feature in `commonMain/.../features/<name>` often has a same-named counterpart under `desktopMain/.../features/<name>`. Entry point is `desktopMain/.../Main.kt` (`com.nuvio.app.MainKt`); the shared shell is `App.kt` / `MainAppContent.kt` / `RootTabHost.kt`. Packages: `com.nuvio.app.core` (auth, network, storage, sync, i18n, ui) and `com.nuvio.app.features` (addons, plugins, player, streams, downloads, trakt/simkl, debrid, p2p, updater, …).

**Generated config**: `GenerateRuntimeConfigsTask` in `composeApp/build.gradle.kts` writes `SupabaseConfig`, `SentryConfig`, `TmdbConfig`, `TraktConfig`, `SimklConfig`, etc. into generated sources from Gradle properties, env vars and an untracked `local.properties` (Trakt/Simkl/MDBList/Premiumize IDs, API URLs, …). Missing keys become empty strings, so those integrations silently do nothing in local builds. Don't hand-edit or look for these classes in the source tree.

**Native player bridges**: playback is libmpv behind a per-OS JNI/native bridge in `desktopMain/native/{windows,macos,linux}/player_bridge.*`, built by host-specific Gradle tasks (`buildWindowsPlayerBridge`, `buildMacosPlayerBridge`, `buildLinuxPlayerBridge`), so each needs its toolchain:
- macOS: bundled `libmpv.2.dylib` + headers under `desktopMain/native/macos/` (per arch); the build fails with a list of missing inputs.
- Linux: system libmpv, `pkg-config`, webkit2gtk-4.1, gtk3, X11 dev packages (see `scripts/linux/install-desktop-build-dependencies.sh`).
- Windows: WebView2 (NuGet) and `libmpv-2.dll`; pass `-Pnuvio.windows.libmpv.runtimeDir=<dir>` if not bundled.
These bridge tasks are not configuration-cache compatible. A `TorrServer` binary per OS is bundled from `desktopMain/torrserver/` for P2P.

**Vendored code**: `vendor/compose-media-player` is included as `:composeMediaPlayer` in `settings.gradle.kts` on non-Windows hosts only. `vendor/quickjs-kt` is used for the plugin runtime. Prebuilt Android AARs sit in `composeApp/libs/`.

**Packaging**: `compose.desktop.application` in `composeApp/build.gradle.kts` defines jlink modules, JVM args (`--add-opens` flags and GTK settings the native bridges depend on), the `nuvio://` and `stremio://` URL schemes on macOS, and per-OS installers. Linux post-processing and verification scripts are in `scripts/linux/`.

**Localization**: strings are Compose resources in `commonMain/composeResources/values*/` (about 25 locales); `Res` accessors come from the generated `nuvio.composeapp.generated.resources` package.

## Contribution policy (enforced by maintainers, see CONTRIBUTING.md)

PRs are limited to reproducible bug fixes, UI glitch fixes with before/after proof, small maintenance, doc fixes and translations. No cosmetic UI changes, unapproved behavior changes, drive-by refactors, or new dependencies/architecture changes without an approved feature-request issue. Keep changes small and single-purpose, and fill the PR template completely or the PR is closed.

## Gotchas

- `composeApp/hs_err_pid19756.log` is a stray JVM crash log committed to the repo; ignore it.
- `org.gradle.configuration-cache` and build cache are on, and heap settings are large (12 GB Gradle, 8 GB Kotlin daemon, 16 GB Kotlin/Native).
