## Context

NuvioMobile (`repos/NuvioMobile`, Kotlin Multiplatform: `composeApp` commonMain plus androidMain/iosMain) has no ViewModels; state lives in repositories. The stream path is:

`StreamParser.parse` (`features/streams/StreamParser.kt:17-64`) -> `StreamItem` -> `StreamsScreen` -> `openSelectedStream` in `StreamDestination.kt` -> either `openExternalStreamUrl` (browser, when `shouldOpenExternally`) or `PlayerLaunch` -> `openExternalPlayback` / `PlayerRoute`.

`StreamParser` (`:33`) drops streams that lack `url`/`infoHash`/`externalUrl`, so `ytId` streams vanish. The app already contains `TrailerPlaybackResolver` (expect in commonMain; `fullCommonMain` actual delegates to `InAppYouTubeExtractor`, store variants return null), `PlayerLaunch.sourceAudioUrl`, and `PlatformPlayerSurface(useYoutubeChunkedPlayback = ...)`, all used for trailers.

## Goals / Non-Goals

**Goals:**
- Make YouTube streams parseable, selectable and playable in the internal player on Android and iOS.
- Respect the external-player setting.
- Reuse the trailer resolver and player plumbing.

**Non-Goals:**
- No server-side or yt-dlp resolution.
- No new extractor logic; extractor robustness (age-gated content) is out of scope.
- No change to trailers.

## Decisions

1. **Model `ytId` explicitly** on `StreamItem` and add an `isYouTube` helper (true for `ytId` or YouTube `externalUrl`). Update the "has playable source" checks (`StreamModels.kt:113`, `:176`) and card visibility (`StreamsScreen.kt:~874`) so such streams are listed. Alternative: rewrite `ytId` to a watch URL in `externalUrl` at parse time; rejected because it hides intent and would collide with the browser fallback.
2. **Resolve in `openSelectedStream` before the `shouldOpenExternally` branch**, mirroring how `DirectDebridPlaybackResolver` resolves then re-enters the function, and in both autoplay copies. To avoid divergence, extract one helper used by all three call sites.
3. **Resolver = `TrailerPlaybackResolver.resolveFromYouTubeUrl(https://www.youtube.com/watch?v=<id>)`.** It already returns `TrailerPlaybackSource(videoUrl, audioUrl?)`, which maps to `PlayerLaunch.sourceUrl` / `sourceAudioUrl`. It has a 10-minute cache, so repeated resolution is cheap.
4. **External player:** `ExternalPlayerPlaybackRequest` has no audio field. If the resolved source has a separate audio URL, use the internal player; if it is muxed/HLS, pass it to the external player. Adding audio-URL support to external launch is a possible follow-up.
5. **Add `useYoutubeChunkedPlayback` to `PlayerLaunch` and `PlayerScreenArgs`**, set for resolved YouTube sources; `PlayerScreen` forwards it to `PlatformPlayerSurface`. iOS (libmpv via MPVKit) ignores it.
6. **Do not write the resolved URL to `StreamLinkCacheRepository`.** Resolved googlevideo URLs expire within hours.
7. **Store variants** (resolver returns null): keep today's behavior, opening `externalUrl` when present, and show an error otherwise.

## Risks / Trade-offs

- [Store builds cannot resolve] -> unchanged behavior, explicit error for `ytId`-only streams.
- [Extractor fragility] -> surface an error; the user can pick another stream.
- [iOS libmpv throttling without chunking] -> verify on device; mitigation is to prefer HLS.
- [Duplicate autoplay logic drifts] -> single shared helper.
- [iOS `full` resolver actual not verified] -> check `iosFull`/`iosMain` sources during implementation; add one if missing.

## Open Questions

- Should external launch gain an audio-URL field, or is "internal when audio is separate" acceptable?
- Is a `yt_id:` prefix present in real add-on responses, or only in meta ids? Confirm against a running YouTubio instance.
