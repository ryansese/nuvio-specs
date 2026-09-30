## Context

NuvioTV (`repos/NuvioTV`, Compose for TV, Media3, Hilt) already plays YouTube streams in-app when they carry a `ytId`:

`StreamScreenViewModel.resolveStreamForPlayback` -> `Stream.youTubeIdToResolve()` -> `YouTubeStreamResolver.resolve()` -> `getStreamForPlayback(resolved, saveLastLink = false)` -> `routePlayback` (`StreamScreen.kt`) -> `Screen.Player`.

`YouTubeStreamResolver` uses `data/trailer/InAppYouTubeExtractor.extractSingleUrl` and is gated by `AppFeaturePolicy.inAppTrailerPlaybackEnabled` (true in `full`, false in `playstore`).

A stream with only a YouTube `externalUrl` satisfies `Stream.isExternal()` and is opened via `ACTION_VIEW` (`openExternalInBrowser`) before any preference is consulted (`routePlayback`, `routeAutoPlay`). YouTubio returns per video `YT-DLP Player <res>` (direct `url`), `Stremio Player` (`ytId`), `External Player` (`externalUrl` = watch URL), the new `Nuvio Player` (`externalUrl` = watch URL) and channel entries.

## Goals / Non-Goals

**Goals:**
- Play a `Nuvio Player` stream in the built-in player from the normal list.
- Prefer 1080p, else the next lower available resolution.
- Preserve `playstore` behavior and every other card.

**Non-Goals:**
- A new tab, group or chip; changing or renaming other cards.
- Changing autoplay, in-player source/episode switching or the preference semantics of other cards.
- A new extractor, yt-dlp or server dependency; reuse `InAppYouTubeExtractor`.
- Separate video+audio (DASH); playback stays a single muxed/HLS URL.
- Changing the trailer player, `ytId` handling or the plugin system.

## Decisions

1. **Recognise by name and URL.** Add `Stream.isNuvioPlayer()` (in `domain/model/Stream.kt`): `url` is null, no torrent/debrid, `name == "Nuvio Player"` and `externalUrl` has host `youtu.be`, `youtube.com` or `*.youtube.com` (after dropping `www.`, the same host rule as `TrailerService.extractYouTubeVideoId`, since `InAppYouTubeExtractor.extractVideoId` accepts any host) and yields a valid id from it (watch, shorts, embed, live, `youtu.be` only). `isExternal()` returns false for it, so `openExternalInBrowser` is skipped; other externals are unchanged. `StreamYouTubeTest` covers it.
2. **Reuse the `ytId` pipeline.** `youTubeIdToResolve()` also returns the video id for a Nuvio Player stream, so `resolveStreamForPlayback` -> `YouTubeStreamResolver.resolve` -> `getStreamForPlayback(saveLastLink = false)` -> `routePlayback` is reused, with the existing `youtube_resolving_stream` indicator and `youtube_resolution_failed` message. The player preference is not consulted.
3. **1080p preference.** `YouTubeStreamResolver` and `InAppYouTubeExtractor.extractSingleUrl` gain an optional maximum height: candidates at or below the cap are kept, otherwise the lowest are used. Only Nuvio Player streams pass 1080; `ytId` streams and trailers do not.
4. **Autoplay excluded.** `StreamAutoPlaySelector` must not select a Nuvio Player stream (it was never selectable as an external card).
5. **Flavor gating on `AppFeaturePolicy`** (`FEATURE_*` flags), not runtime flavor checks. Where in-app playback is disabled, `isNuvioPlayer()` is not applied, so the card opens the watch page like other externals.

## Risks / Trade-offs

- [Recognition depends on the exact name `Nuvio Player`] -> documented in `youtube-nuvio-player-stream-youtubio`; only this name is special-cased.
- [Extractor breakage as YouTube changes] -> failure surfaces `youtube_resolution_failed`; the user can pick another source.
- [Single muxed URL caps quality] -> accepted; 1080p applies within what muxed/HLS sources offer.
- [Resolution latency] -> show the existing resolving indicator.
- [Channel pages or other YouTube URLs] -> only video URL forms with a valid id qualify.
