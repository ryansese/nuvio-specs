---
paths:
  - "repos/NuvioTV/**"
---

# NuvioTV (repos/NuvioTV)

This file provides guidance to Claude Code (claude.ai/code) when working with code in the NuvioTV submodule.

## What this is

NuvioTV: an Android TV media app (Kotlin, Jetpack Compose for TV / TV Material 3, Media3, Hilt). It aggregates Stremio-compatible add-ons into a library with playback, Trakt/Simkl/MDBList integrations, and Supabase-backed account sync. Single app module (`:app`, package `com.nuvio.tv`), plus `:baselineprofile` and `:ffmpeg-decoder-downmix`. `libmpv-android/` and `DV7/` (libdovi prebuilts for Dolby Vision profile 7 → 8.1) are supporting native pieces.

## Commands

```bash
./gradlew :app:assembleFullDebug                       # main dev build (also what CI runs)
./gradlew :app:testFullDebugUnitTest                   # unit tests (src/test, src/testFull)
./gradlew :app:testFullDebugUnitTest --tests 'com.nuvio.tv.updater.*'   # single package/class (CI runs this filter)
python3 -m unittest discover scripts/tests             # release-notes / release-channel script tests
```

- Copy `local.example.properties` to `local.properties` (gitignored) and fill in Supabase/Trakt/Simkl/MDBList etc. values; `app/build.gradle.kts` reads them into `BuildConfig`. DV7 native build and local ffmpeg decoder are toggled by properties in that file (`DOVI_NATIVE_ENABLED`, `USE_LOCAL_FFMPEG_DECODER`).
- Debug builds are signed with the release keystore config (`nuviotv.jks`, gitignored); `debuggable` is a Gradle property (`-Pdebuggable=true`).
- No lint task is wired into CI.

## Architecture

Layered under `app/src/main/java/com/nuvio/tv/`:

- `core/` — cross-cutting services organised by feature (`player`, `streams`, `plugin`, `sync`, `auth`, `profile`, `debrid`, `torrent`, `trakt`, `tmdb`, `network`, `di`, …). Hilt modules live in `core/di/`.
- `data/` — `local/` (DataStore-based preferences, one class per concern), `remote/` (API clients), `repository/` implementations, `mapper/`, plus `mdblist`, `simkl`, `trailer`.
- `domain/` — models and repository interfaces.
- `ui/` — `navigation/` (`Screen.kt`, `NuvioNavHost.kt`), `screens/`, `components/`, `theme/`. TV focus/remote navigation is a first-class concern.
- `MainActivity.kt` gates startup: account QR sign-in → profile selection (PIN) → layout choice → main shell (Home, Search, Library, Add-ons, Settings).

Two experience modes (Essential vs Advanced) are described in `docs/essential-mode.md`; state is in `ExperienceModeDataStore`.

### Product flavors (`distribution` dimension)

- `full` — GitHub/APK build: plugins, in-app updater, in-app trailers, custom server connections enabled.
- `playstore` — `applicationId com.nuvio.app`; those features are disabled via `BuildConfig.FEATURE_*` flags.

Source sets `src/full` and `src/playstore` hold flavor-specific code; gate behaviour on the `FEATURE_*` flags rather than checking the flavor at runtime. Localised strings live in many `res/values-*` directories, so a new string in `values/` needs a corresponding translation pass only if the task calls for it.

## Contribution rules (from CONTRIBUTING.md, enforced strictly)

PRs are accepted only for reproducible bug fixes, documented UI glitches, small maintenance, doc fixes, translations, or work tied to an approved feature request. No cosmetic/"polish" UI changes, unapproved features, refactors without a maintenance need, or new dependencies/architecture changes. UI fixes need a linked bug plus before/after evidence; behavior changes (playback, source selection, resume/watched state, sync, focus, etc.) must explain old vs new behavior and how they were tested. Keep changes minimal.
