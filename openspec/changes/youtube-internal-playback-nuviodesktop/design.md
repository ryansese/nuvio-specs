## Context

NuvioDesktop (`repos/NuvioDesktop`, Kotlin Multiplatform / Compose Desktop) shares the commonMain layout of NuvioMobile (`StreamParser`, `StreamItem`, `StreamDestination.openSelectedStream`, `PlayerLaunch`, and the trailer resolver, with `fullCommonMain` sources compiled into `desktopMain`).

`StreamParser.parse` (`features/streams/StreamParser.kt`) -> `StreamItem` -> `StreamsRepository` groups (`AddonStreamGroup` per add-on) -> `StreamsScreen` -> `openSelectedStream` in `StreamDestination.kt` -> either `openExternalStreamUrl` (browser, when `shouldOpenExternally`) or `PlayerLaunch` -> `openExternalPlayback` / `PlayerRoute`.

YouTubio streams for a video come from embedded streams parsed by `MetaDetailsParser`: `YT-DLP Player <res>` (direct `url`), `External Player` and the new `Nuvio Player` (`externalUrl` = YouTube watch URL), channel entries (`externalUrl`, not a video). `ytId`-only entries (`Stremio Player`) are dropped by the parsers today; that stays as it is.

`TrailerPlaybackResolver` (expect in commonMain; `fullCommonMain` actual delegates to `InAppYouTubeExtractor`, store variants return null), `PlayerLaunch.sourceAudioUrl` and `PlatformPlayerSurface(useYoutubeChunkedPlayback = ...)` already exist for trailers. The extractor picks the tallest stream (`videoScore` is height-first).

Findings from running the app against a local YouTubio add-on:
- The list comes from **embedded streams** parsed by `MetaDetailsParser.embeddedStreams`, which drops `ytId`-only streams and names the whole group after the first stream. That behavior is existing and is left alone.
- The desktop internal player is libmpv behind per-OS JNI bridges (`desktopMain/native/{macos,linux,windows}`), driven by `NativePlayerController`. `PlatformPlayerSurface` receives `sourceAudioUrl`, but the desktop engine did not forward it and the bridges had no audio parameter.
- On macOS `mpv_create` fails ("Non-C locale detected") unless `LC_NUMERIC` is `C`; the Linux bridge already calls `setlocale(LC_NUMERIC, "C")`, macOS and Windows do not. The player screen stays on its loading animation when this happens.
- libmpv's list option for external audio is `audio-files` (there is no `audio-file`). Its string form splits on `:` (`;` on Windows), so a URL cannot be passed as a string; a one-item node array through `mpv_set_option(..., MPV_FORMAT_NODE, ...)` before `mpv_initialize` keeps it intact.
- `AppFeaturePolicy.desktop` sets `externalPlayerSupported = false`; Windows also sets `trailerPlaybackMode = EXTERNAL`.
- The extractor does not decode YouTube's `n` throttling token; Android works around throttling with `YoutubeChunkedDataSourceFactory`, desktop has no equivalent.

## Goals / Non-Goals

**Goals:**
- Play a `Nuvio Player` stream in the built-in player from the normal list on macOS, Windows and Linux.
- Prefer 1080p, else the next lower available resolution.
- Audible playback when the resolved source has separate audio, with no stalling from throttled downloads.

**Non-Goals:**
- A new tab or group; changing or renaming other cards.
- Supporting `ytId`-only streams (`Stremio Player`); they remain unlisted.
- Changing autoplay, enabling the external player on desktop, bundling yt-dlp, or changing trailers.

## Decisions

1. **Recognise by name and URL.** `StreamItem.isNuvioPlayer` (in `features/streams/StreamModels.kt`) is true when `url` is null, `name == "Nuvio Player"` and `externalUrl` is a YouTube watch, shorts, embed, live or `youtu.be` URL with a valid id; add `youtubeVideoId` / `youtubeWatchUrl` helpers. `shouldOpenExternally` is false for it; every other stream is unchanged and `StreamParser` and `MetaDetailsParser` are untouched.
2. **Resolution and routing.** `openSelectedStream` resolves a Nuvio Player stream through `TrailerPlaybackResolver.resolveFromYouTubeUrl(url, maxHeight = 1080)` into a `PlayerLaunch` before the `shouldOpenExternally` branch, always opens the internal player, shows an error on failure, and never writes to `StreamLinkCacheRepository`. The resolver has a 10-minute cache whose key includes the cap.
3. **1080p preference.** `resolveFromYouTubeUrl` and the extractor gain an optional `maxHeight`: candidates at or below the cap are kept, otherwise the lowest are used. Store actuals accept and ignore it. Trailers pass no cap.
4. **Autoplay excluded.** `StreamAutoPlaySelector` must not select a Nuvio Player stream.
5. **Audio forwarding.** Thread `sourceAudioUrl` through `PlayerEngine.desktop.kt` -> `NativePlayerController.attach` -> `NativePlayerBridge.create(audioUrl)` -> each bridge, which sets `audio-files` as a node array before initialize. A muxed or HLS source needs no audio URL.
6. **macOS locale.** Call `setlocale(LC_NUMERIC, "C")` before `mpv_create` in the macOS bridge; the same on Windows if `mpv_create` fails there.
7. **Throttling mitigation** through preferring an HLS source, a local range-chunking proxy, or mpv options; decide after measuring a long video.
8. **External-player logic is ignored** for Nuvio Player; the internal path is unconditional, and Windows' trailer EXTERNAL mode must not affect stream playback.
9. **Revert the earlier approach first.** Commit `6c4aba22` (parses `ytId`, plays YouTube `externalUrl` streams inline for any name, changes autoplay, uses a non-existent `audio-file` option) conflicts with this design and is reworked by the tasks below.

## Risks / Trade-offs

- [Recognition depends on the exact name `Nuvio Player`] -> documented in `youtube-nuvio-player-stream-youtubio`.
- [Native bridge changes on three platforms] -> Windows and Linux are not buildable on a macOS host; verify each OS.
- [Throttled playback] -> measure; prefer HLS, chunk requests, or adjust mpv options.
- [Extractor fragility] -> surface an error message.
- [Drift between Mobile and Desktop commonMain] -> keep shared logic in one helper and mirror tests.

## Open Questions

- Which throttling mitigation is needed after measuring a video longer than 10 minutes?
