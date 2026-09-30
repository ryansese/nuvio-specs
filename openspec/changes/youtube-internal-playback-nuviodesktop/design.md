## Context

NuvioDesktop (`repos/NuvioDesktop`, Kotlin Multiplatform / Compose Desktop) shares the commonMain layout of NuvioMobile: `StreamParser`, `StreamItem`, `StreamDestination.openSelectedStream`, `PlayerLaunch`, and the trailer resolver (`fullCommonMain` sources compiled into `desktopMain` via `build.gradle.kts`).

Findings from running the app against a local YouTubio add-on:
- YouTubio returns, per video, `YT-DLP Player <res>` (direct `url`), `Stremio Player` (`ytId` only), `External Player` (`externalUrl` = YouTube watch URL), `YT-DLP Channel` and `External Channel` (`externalUrl`, not a video).
- The list comes from **embedded streams** parsed by `MetaDetailsParser.embeddedStreams`, not `StreamParser`. It drops `ytId`-only streams and names the whole group after the first stream (`embeddedStreams.first().addonName`), which is why the group is titled `YT-DLP Player 640x360`. That behavior is existing and is left alone.
- Tabs on the streams screen are the per-add-on groups (`ProviderFilterRow` renders one chip per `AddonStreamGroup`; `StreamsUiState.filteredGroups` filters by `selectedFilter`).
- The desktop internal player is libmpv behind per-OS JNI bridges (`desktopMain/native/{macos,linux,windows}`), driven by `NativePlayerController`. `PlatformPlayerSurface` receives `sourceAudioUrl`, but the desktop engine did not forward it and the bridges had no audio parameter.
- On macOS `mpv_create` fails ("Non-C locale detected") unless `LC_NUMERIC` is `C`; the Linux bridge already calls `setlocale(LC_NUMERIC, "C")`, macOS and Windows do not. The player screen stays on its loading animation when this happens.
- libmpv's list option for external audio is `audio-files` (there is no `audio-file`). Its string form splits on `:` (`;` on Windows) and ignores `%n%` quoting, so a URL cannot be passed as a string. Passing a one-item node array through `mpv_set_option(..., MPV_FORMAT_NODE, ...)` before `mpv_initialize` keeps the URL intact.
- `AppFeaturePolicy.desktop` sets `externalPlayerSupported = false`; Windows also sets `trailerPlaybackMode = EXTERNAL`.
- The extractor picks the tallest stream (`videoScore` is height-first) and does not decode YouTube's `n` throttling token; Android works around throttling with `YoutubeChunkedDataSourceFactory`, desktop has no equivalent.

## Goals / Non-Goals

**Goals:**
- Leave the existing stream lists exactly as they are.
- Offer in-app playback of YouTube results in a separate **Internal Player** tab on macOS, Windows and Linux.
- Prefer 1080p, else the next lower available resolution, for in-app YouTube playback.
- Audible playback when the resolved source has separate audio.
- No stalling from throttled googlevideo downloads.

**Non-Goals:**
- Changing, renaming, reordering or filtering existing groups and cards.
- Supporting `ytId`-only streams (`Stremio Player`); they remain unlisted.
- Changing autoplay.
- Enabling the external player on desktop.
- A bundled yt-dlp; it stays a documented alternative resolver.
- Changing trailer behavior (including its resolution).

## Decisions

1. **Derived group, not a data change.** `StreamsUiState.displayGroups` returns the original `groups` plus, when applicable, one extra `AddonStreamGroup` (`addonId = "nuvio:internal-player"`, name `Internal Player`). `filteredGroups` and the filter rows read `displayGroups`; repositories, autoplay and the in-player sources panel keep reading `groups`.
2. **Tab contents.** For every group holding a YouTube `externalUrl` stream (watch, shorts, embed, `live`, `youtu.be`): its direct-URL streams are copied into the tab as they are, and each YouTube stream is added once per add-on and video id as a copy flagged `playInternally = true`. Originals are never removed. The tab is absent when neither exists.
3. **Per-stream flag.** `StreamItem.playInternally` (default false) makes `isYouTube` true only for the tab's copies; `shouldOpenExternally` is false only for those copies. Originals keep opening the browser.
4. **Resolution and routing.** `openSelectedStream` resolves a `playInternally` stream through `TrailerPlaybackResolver.resolveFromYouTubeUrl(url, maxHeight = 1080)` into a `PlayerLaunch` (video URL plus optional audio URL) before the external branch, shows a toast on failure, and never writes to `StreamLinkCacheRepository`.
5. **1080p preference.** `resolveFromYouTubeUrl` and the extractor gain an optional `maxHeight`. Candidates (progressive, adaptive video, HLS variants) at or below the cap are kept; if none fit, the lowest available are used. The resolver cache key includes the cap. Trailers pass no cap.
6. **Audio forwarding.** Thread `sourceAudioUrl` through `PlayerEngine.desktop.kt` -> `NativePlayerController.attach` -> `NativePlayerBridge.create(audioUrl)` -> each bridge, which sets `audio-files` as a node array before initialize. A muxed or HLS source needs no audio URL.
7. **macOS locale.** Call `setlocale(LC_NUMERIC, "C")` before `mpv_create` in the macOS bridge; the same is needed on Windows if `mpv_create` fails there.
8. **Throttling mitigation** through preferring an HLS source, a local range-chunking proxy, or mpv options; decide after measuring with a long video. There is no desktop equivalent of `YoutubeChunkedDataSourceFactory`.
9. **External-player logic is ignored** for the tab's entries; the internal path is unconditional, and Windows' trailer EXTERNAL mode must not affect stream playback.

## Risks / Trade-offs

- [Direct results appear in both the original group and the tab] -> intentional so the old list is unchanged; revisit if it confuses users.
- [Moving direct results by rule (any group with a YouTube `externalUrl` stream) may catch other add-ons] -> limited to groups that also contain a YouTube video link.
- [Native bridge changes on three platforms] -> Windows and Linux are not buildable on a macOS host; verify each OS.
- [Throttled playback] -> measure; prefer HLS, chunk requests, or adjust mpv options.
- [Extractor fragility] -> surface an error message.
- [Drift between Mobile and Desktop commonMain] -> keep shared logic in one helper and mirror tests.

## Open Questions

- Should the tab's copy of `External Player` keep that name, or be relabelled?
- Should direct `YT-DLP Player` results be moved out of the original group (list changes) instead of copied?
- Which throttling mitigation is needed after measuring a video longer than 10 minutes?
