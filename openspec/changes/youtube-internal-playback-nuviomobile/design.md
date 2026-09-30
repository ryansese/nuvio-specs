## Context

NuvioMobile (`repos/NuvioMobile`, Kotlin Multiplatform: `composeApp` commonMain plus androidMain/iosMain) has no ViewModels; state lives in repositories. It shares its streams layout with NuvioDesktop. The stream path is:

`StreamParser.parse` (`features/streams/StreamParser.kt`) -> `StreamItem` -> `StreamsRepository` groups (`AddonStreamGroup` per add-on) -> `StreamsScreen` -> `openSelectedStream` in `StreamDestination.kt` -> either `openExternalStreamUrl` (browser, when `shouldOpenExternally`) or `PlayerLaunch` -> `openExternalPlayback` / `PlayerRoute`.

YouTubio streams for a video come from embedded streams parsed by `MetaDetailsParser`: `YT-DLP Player <res>` (direct `url`), `External Player` and the new `Nuvio Player` (`externalUrl` = YouTube watch URL), channel entries (`externalUrl`, not a video). `ytId`-only entries (`Stremio Player`) are dropped by the parsers today; that stays as it is.

`TrailerPlaybackResolver` (expect in commonMain; `fullCommonMain` actual delegates to `InAppYouTubeExtractor`, store variants return null), `PlayerLaunch.sourceAudioUrl` and `PlatformPlayerSurface(useYoutubeChunkedPlayback = ...)` already exist for trailers. The extractor picks the tallest stream (`videoScore` is height-first).

## Goals / Non-Goals

**Goals:**
- Play a `Nuvio Player` stream in the built-in player from the normal list on Android and iOS.
- Prefer 1080p, else the next lower available resolution.
- Reuse the trailer resolver and player plumbing.

**Non-Goals:**
- A new tab or group; changing or renaming other cards.
- Supporting `ytId`-only streams; they remain unlisted.
- Changing autoplay or the external-player behavior of other cards.
- Server-side or yt-dlp resolution, or new extractor logic beyond the cap.
- Changing trailers.

## Decisions

1. **Recognise by name and URL.** `StreamItem.isNuvioPlayer` (in `features/streams/StreamModels.kt`) is true when `url` is null, `name == "Nuvio Player"` and `externalUrl` is a URL on host `youtu.be`, `youtube.com` or a `*.youtube.com` subdomain (after dropping `www.`) with a watch, shorts, embed, live or `youtu.be` path and a valid video id; add `youtubeVideoId` / `youtubeWatchUrl` helpers. `shouldOpenExternally` is false for it; every other stream is unchanged and `StreamParser` and `MetaDetailsParser` are untouched.
2. **Resolution and routing.** `openSelectedStream` resolves a Nuvio Player stream through `TrailerPlaybackResolver.resolveFromYouTubeUrl(url, maxHeight = 1080)` into a `PlayerLaunch` before the `shouldOpenExternally` branch, always opens the internal player, shows an error on failure, and never writes to `StreamLinkCacheRepository`. The resolver has a 10-minute cache whose key includes the cap.
3. **1080p preference.** `resolveFromYouTubeUrl` and the extractor gain an optional `maxHeight`: candidates at or below the cap are kept, otherwise the lowest are used. Store actuals accept and ignore it. Trailers pass no cap.
4. **Autoplay excluded.** `StreamAutoPlaySelector` must not select a Nuvio Player stream.
5. **Add `useYoutubeChunkedPlayback` to `PlayerLaunch` and `PlayerScreenArgs`**, set for resolved YouTube sources; `PlayerScreen` forwards it to `PlatformPlayerSurface`. iOS (libmpv via MPVKit) ignores it.
6. **Store variants** (resolver returns null): `Nuvio Player` behaves like any other YouTube external card.

## Risks / Trade-offs

- [Recognition depends on the exact name `Nuvio Player`] -> documented in `youtube-nuvio-player-stream-youtubio`.
- [Extractor fragility] -> surface an error; the user can pick another stream.
- [iOS libmpv throttling without chunking, and separate audio on iOS] -> verify on device; mitigation is to prefer HLS.
- [iOS `full` resolver actual not verified] -> check `iosFull`/`iosMain` sources during implementation; add one if missing.
