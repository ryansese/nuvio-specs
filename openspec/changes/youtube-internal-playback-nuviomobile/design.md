## Context

NuvioMobile (`repos/NuvioMobile`, Kotlin Multiplatform: `composeApp` commonMain plus androidMain/iosMain) has no ViewModels; state lives in repositories. It shares its streams layout with NuvioDesktop. The stream path is:

`StreamParser.parse` (`features/streams/StreamParser.kt`) -> `StreamItem` -> `StreamsRepository` groups (`AddonStreamGroup` per add-on) -> `StreamsScreen` (`ProviderFilterRow` renders one chip per group; `StreamsUiState.filteredGroups` filters by `selectedFilter`) -> `openSelectedStream` in `StreamDestination.kt` -> either `openExternalStreamUrl` (browser, when `shouldOpenExternally`) or `PlayerLaunch` -> `openExternalPlayback` / `PlayerRoute`.

For an add-on such as YouTubio, per-video streams come from embedded streams parsed by `MetaDetailsParser`: `YT-DLP Player <res>` (direct `url`), `External Player` (`externalUrl` = YouTube watch URL), `YT-DLP Channel` / `External Channel` (`externalUrl`, not a video). `ytId`-only entries (`Stremio Player`) are dropped by the parsers today, and the embedded group is titled after its first stream. All of that is existing behavior and is left alone.

The app already contains `TrailerPlaybackResolver` (expect in commonMain; `fullCommonMain` actual delegates to `InAppYouTubeExtractor`, store variants return null), `PlayerLaunch.sourceAudioUrl`, and `PlatformPlayerSurface(useYoutubeChunkedPlayback = ...)`, all used for trailers. The extractor picks the tallest stream (`videoScore` is height-first).

## Goals / Non-Goals

**Goals:**
- Leave the existing stream lists exactly as they are.
- Offer in-app playback of YouTube results in a separate **Internal Player** tab on Android and iOS.
- Prefer 1080p, else the next lower available resolution, for in-app YouTube playback.
- Reuse the trailer resolver and player plumbing.

**Non-Goals:**
- Changing, renaming, reordering or filtering existing groups and cards.
- Supporting `ytId`-only streams; they remain unlisted.
- Changing autoplay or the external-player behavior of existing cards.
- Server-side or yt-dlp resolution, or new extractor logic beyond the resolution cap.
- Changing trailers.

## Decisions

1. **Derived group, not a data change.** `StreamsUiState.displayGroups` returns the original `groups` plus, when applicable, one extra `AddonStreamGroup` (`addonId = "nuvio:internal-player"`, name `Internal Player`). `filteredGroups` and the `ProviderFilterRow` call sites (phone and tablet layouts) read `displayGroups`; repositories and autoplay keep reading `groups`.
2. **Tab contents.** For every group holding a YouTube `externalUrl` stream (watch, shorts, embed, `live`, `youtu.be`): its direct-URL streams are copied into the tab as they are, and each YouTube stream is added once per add-on and video id as a copy flagged `playInternally = true`. Originals are never removed. The tab is absent when neither exists or when the build has no resolver.
3. **Per-stream flag.** `StreamItem.playInternally` (default false) makes `isYouTube` true only for the tab's copies; `shouldOpenExternally` is false only for those copies.
4. **Resolution and routing.** `openSelectedStream` resolves a `playInternally` stream through `TrailerPlaybackResolver.resolveFromYouTubeUrl(url, maxHeight = 1080)` into a `PlayerLaunch` (`sourceUrl` / `sourceAudioUrl`) before the `shouldOpenExternally` branch, always opens the internal player, shows an error on failure, and never writes to `StreamLinkCacheRepository`. The resolver has a 10-minute cache; the cap is part of its key.
5. **1080p preference.** `resolveFromYouTubeUrl` and the extractor gain an optional `maxHeight`: candidates at or below the cap are kept; if none fit, the lowest available are used. The `iosAppStore` and `androidPlaystore` actuals accept and ignore it. Trailers pass no cap.
6. **Add `useYoutubeChunkedPlayback` to `PlayerLaunch` and `PlayerScreenArgs`**, set for resolved YouTube sources; `PlayerScreen` forwards it to `PlatformPlayerSurface`. iOS (libmpv via MPVKit) ignores it.
7. **Store variants** (resolver returns null): no Internal Player tab and no other behavior change.

## Risks / Trade-offs

- [Direct results appear in both the original group and the tab] -> intentional so the old list is unchanged; revisit if it confuses users.
- [Copying direct results by rule (any group with a YouTube `externalUrl` stream) may catch other add-ons] -> limited to groups that also contain a YouTube video link.
- [Extractor fragility] -> surface an error; the user can pick another stream.
- [iOS libmpv throttling without chunking, and separate audio on iOS] -> verify on device; mitigation is to prefer HLS.
- [iOS `full` resolver actual not verified] -> check `iosFull`/`iosMain` sources during implementation; add one if missing.

## Open Questions

- Should the tab's copy of `External Player` keep that name, or be relabelled?
- Should direct `YT-DLP Player` results be moved out of the original group (list changes) instead of copied?
- Should the external-player setting ever apply to the tab's entries?
