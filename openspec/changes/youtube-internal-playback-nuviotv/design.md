## Context

NuvioTV (`repos/NuvioTV`, Compose for TV, Media3, Hilt) already plays YouTube streams in-app when they carry a `ytId`:

`StreamScreenViewModel.resolveStreamForPlayback` (`:1153`) -> `Stream.youTubeIdToResolve()` -> `YouTubeStreamResolver.resolve()` -> `getStreamForPlayback(resolved, saveLastLink = false)` -> `routePlayback` (`StreamScreen.kt:195`) -> `Screen.Player`.

`YouTubeStreamResolver` uses `data/trailer/InAppYouTubeExtractor.extractSingleUrl` and is gated by `AppFeaturePolicy.inAppTrailerPlaybackEnabled` (true in `full`, false in `playstore`, where it sets `externalUrl` to the watch page instead).

The gap: a stream with only a YouTube `externalUrl` satisfies `Stream.isExternal()` and is opened via `ACTION_VIEW` (`openExternalInBrowser`) before any preference is consulted.

## Goals / Non-Goals

**Goals:**
- Route YouTube `externalUrl` streams through the existing resolve-then-play path.
- Keep one detection point so every caller (stream list, autoplay, in-player switching) behaves the same.
- Preserve player-preference semantics and `playstore` behavior.

**Non-Goals:**
- No new extractor and no yt-dlp/server dependency; reuse `InAppYouTubeExtractor`.
- No change to the trailer player or plugin system.
- No separate video+audio (DASH) support; playback stays a single muxed/HLS URL.

## Decisions

1. **Detect in the domain model, not in UI callers.** Extend `Stream.youTubeIdToResolve()` to derive the id from a YouTube `externalUrl` (via `InAppYouTubeExtractor.extractVideoId`) and make `isExternal()` false for those. Alternative considered: special-casing inside `openExternalInBrowser`; rejected because autoplay and in-player switching would each need the same patch. `StreamYouTubeTest` covers this logic.
2. **Reuse `YouTubeStreamResolver` unchanged.** It already handles the flavor flag and returns the stream with `url` set. In `playstore`, `resolve()` keeps returning a stream with `externalUrl` set, so the browser fallback remains.
3. **Keep flavor gating on `AppFeaturePolicy`** rather than runtime flavor checks, per the repo convention (`FEATURE_*` flags).
4. **Autoplay eligibility.** `StreamAutoPlaySelector.kt:35` excludes external streams. Since detected YouTube streams stop being `isExternal()`, they become eligible with no selector change; verify with a test and avoid autoplaying on resolution latency assumptions.
5. **Never save resolved URLs as the last link** (already `saveLastLink = false`); the same applies to the newly covered streams.

## Risks / Trade-offs

- [Extractor breakage as YouTube changes] -> failure surfaces `youtube_resolution_failed`; the user can still pick another source. Longer-term option: add a server-side fallback (out of scope).
- [Single muxed URL caps quality] -> accepted for now; documented as a limitation.
- [Resolution latency delays playback] -> show the existing `youtube_resolving_stream` indicator.
- [False positives for arbitrary `youtube.com` URLs, e.g. channel pages] -> only watch, shorts, embed and `youtu.be` forms with a valid 11-char id are treated as resolvable.

## Open Questions

- Should a resolved stream be retried through the external browser automatically on failure in the `full` flavor? Default: no, show the error.
