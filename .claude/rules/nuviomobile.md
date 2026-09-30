---
paths:
  - "repos/NuvioMobile/**"
---

# NuvioMobile (repos/NuvioMobile)

This file provides guidance to Claude Code (claude.ai/code) when working with code in the NuvioMobile submodule.

## Overview

Nuvio Mobile is a Kotlin Multiplatform + Compose Multiplatform app (Android and iOS) that aggregates user-supplied Stremio-style addons and plugins into a library with playback, sync, and tracking. The active branch is `cmp-rewrite`. This checkout is a submodule of the `nuvio-specs` workspace.

## Build and run

Android (Gradle) and iOS (Xcode, which calls back into Gradle for the shared framework):

```bash
./gradlew :androidApp:assembleFullDebug
env NUVIO_IOS_DISTRIBUTION=full xcodebuild -project iosApp/iosApp.xcodeproj -scheme iosApp \
  -configuration Debug -sdk iphonesimulator -derivedDataPath build/ios-derived-full-simulator \
  CODE_SIGNING_ALLOWED=NO build
```

`./scripts/run-mobile.sh android|ios [e|s|p] [full|playstore|appstore]` builds, installs on emulators/devices/simulators, and launches. `scripts/debug_logs.sh` is for log capture.

The only CI build check is `./gradlew :androidApp:assembleFullDebug -Pnuvio.android.distribution=full -Pnuvio.ios.distribution=appstore` (`.github/workflows/pr-full-debug-build.yml`). No lint task is configured.

### Distribution flavors

The project builds two distributions per platform, and `composeApp/build.gradle.kts` enforces the choice at configuration time:

- Android: `full` or `playstore`. Set with `-Pnuvio.android.distribution=...` or `NUVIO_ANDROID_DISTRIBUTION` in `local.properties`. It is inferred from the task name (`...Full...` / `...Playstore...`). Aggregate tasks such as `assemble` or `build` fail unless the property is set. Building both in one invocation is rejected.
- iOS: `full` or `appstore` (default `appstore`). Set with `-Pnuvio.ios.distribution`, the `NUVIO_IOS_DISTRIBUTION` env var, or `local.properties`.

Source sets differ per distribution. `src/androidFull`, `src/androidPlaystore`, `src/iosFull` and `src/iosAppStore` hold the platform-specific variants. `src/fullCommonMain` is added only for `full` Android builds, and it is where the plugin runtime lives (QuickJS via a bundled `libs/quickjs-kt-*.aar`, plus ksoup). `androidFullHostTest` is likewise only wired in for `full`. The Play Store and App Store builds deliberately omit the plugin runtime, so code touching plugins must work across both variants.

### Runtime config

Secrets and endpoints are not checked in. They come from `local.properties` (gitignored) or environment variables: `NUVIO_SUPABASE_URL`, `NUVIO_SUPABASE_ANON_KEY`, `NUVIO_SUPABASE_FALLBACK_URL`, `SENTRY_DSN`, `TMDB_API_KEY`, among others. The `generateRuntimeConfigs` Gradle task turns them into generated Kotlin under `composeApp/build/generated/runtime-config`, which is added to `commonMain`. The app version comes from `iosApp/Configuration/Version.xcconfig`, for both platforms.

## Architecture

Modules: `:composeApp` (all shared code, compiled as a KMP library) and `:androidApp` (thin Android application shell, flavor dimension `distribution`). `iosApp/` is the Xcode project. It hosts the SwiftUI entry point and native pieces (the `Player/` code, Live Activities for downloads, the orientation lock coordinator), and links MPVKit (the `MPVKit` submodule) and the shared framework.

Shared code is in `composeApp/src/commonMain/kotlin/com/nuvio/app/`:

- `App.kt`, `AppGate*.kt`, `MainAppContent.kt`, `RootTabHost.kt`, and the `*Destinations.kt` files make up the app shell: startup gating, the root tab host, and Navigation 3 destinations.
- `core/` holds cross-cutting infrastructure: `auth`, `network`, `storage`, `sync`, `tracking`, `i18n`, `deeplink`, `build` and others.
- `features/` holds one package per feature: addons, catalog, details, streams, player, downloads, library, search, profiles, plugins, debrid, trakt/simkl/mdblist/tmdb integrations, and more.
- Platform differences use `expect`/`actual`. `Platform.kt` is the canonical example. Most feature packages carry a `*Platform.kt` expect declaration, with actuals in `androidMain` and `iosMain`. Add a declaration in common and an actual for each target, not a platform check in shared code.

The app talks to two kinds of external sources. Stremio-protocol addons are handled in `features/addons`: a manifest parser, an HTTP client and a repository. Nuvio's own scraper plugins are JS run on-device and live in `features/plugins` plus `fullCommonMain`. User accounts, profiles and cross-device sync (watch progress, settings, provider credentials) go through Supabase (`core/auth`, `core/sync`).

Localized strings are Compose resources in `composeApp/src/commonMain/composeResources/values*/` (around 30 locales). Translation changes are welcome upstream, and adding a string means adding it to the default `values/` first.

`composeApp/src/desktopMain` exists, but `composeApp/build.gradle.kts` has no desktop target, so do not assume it builds from this repo. Desktop lives in the separate NuvioDesktop repo.

## Tests

Tests live in `commonTest`, `androidHostTest` (Robolectric and Compose UI tests), `androidFullHostTest`, and `iosTest` under `composeApp/src`, plus instrumented tests in `androidApp/src/androidTest`. CI does not run them. Run host tests with a distribution specified, for example `./gradlew :composeApp:testAndroidHostTest -Pnuvio.android.distribution=full`. The task name was not verified here, so check `./gradlew :composeApp:tasks --all` if it fails.

## Contribution policy (from CONTRIBUTING.md)

Upstream is strict. PRs are accepted only for reproducible bug fixes, documented UI glitches (with before/after proof), translations, and small maintenance work. Cosmetic UI changes, new features, behavior changes without a linked bug, unrequested refactors, and new dependencies need an approved feature-request issue first. Keep changes minimal and focused on one problem, and complete the PR template.
