## Context

NuvioTV (`repos/NuvioTV`, Compose for TV, Media3, Hilt) already plays YouTube streams in-app when they carry a `ytId`:

`StreamScreenViewModel.resolveStreamForPlayback` -> `Stream.youTubeIdToResolve()` -> `YouTubeStreamResolver.resolve()` -> `getStreamForPlayback(resolved, saveLastLink = false)` -> `routePlayback` (`StreamScreen.kt`) -> `Screen.Player`.

`YouTubeStreamResolver` uses `data/trailer/InAppYouTubeExtractor.extractSingleUrl` and is gated by `AppFeaturePolicy.inAppTrailerPlaybackEnabled` (true in `full`, false in `playstore`).

A stream with only a YouTube `externalUrl` satisfies `Stream.isExternal()` and is opened via `ACTION_VIEW` (`openExternalInBrowser`) before any preference is consulted. For YouTubio, per-video streams are `YT-DLP Player <res>` (direct `url`), `External Player` (`externalUrl` = YouTube watch URL) and channel entries; that list and its `ytId` handling are existing behavior and are left alone.

The stream list filters by add-on: `StreamScreenUiState.selectedAddonFilter` (null = All), `StreamScreenEvent.OnAddonFilterSelected`, rendered by `AddonFilterChips`.

## Goals / Non-Goals

**Goals:**
- Leave the existing stream list and `ytId` behavior exactly as they are.
- Offer in-app playback of YouTube `externalUrl` results in a separate **Internal Player** tab.
- Prefer 1080p, else the next lower available resolution, for these streams.
- Preserve `playstore` behavior.

**Non-Goals:**
- Changing, renaming, reordering or filtering existing groups and cards, or changing `isExternal()` for originals.
- Changing autoplay, in-player source/episode switching or the player-preference semantics of existing cards.
- A new extractor, or yt-dlp/server dependency; reuse `InAppYouTubeExtractor`.
- Separate video+audio (DASH) support; playback stays a single muxed/HLS URL.
- Changing the trailer player or plugin system.

## Decisions

1. **Detect in the domain model.** Add a helper on `Stream` that returns the video id when `url` is null and `externalUrl` is a YouTube watch, shorts, embed, live or `youtu.be` URL (via `InAppYouTubeExtractor.extractVideoId`), and a `playInternally` flag (default false). `isExternal()` stays true for originals and is false only for `playInternally` copies. `StreamYouTubeTest` covers this logic.
2. **Derived tab, not a data change.** The UI state exposes the add-on groups as they are plus, when applicable, one extra entry named `Internal Player` in the filter chips. Selecting it shows the derived list; `selectedAddonFilter` and the repositories keep working on real add-on names. Autoplay and the in-player source list keep reading the original streams.
3. **Tab contents.** For every add-on group holding a YouTube `externalUrl` stream: its direct-URL streams are copied into the tab as they are, and each YouTube stream is added once per add-on and video id as a `playInternally` copy. Originals are never removed. The tab is absent when neither exists or when in-app resolution is disabled.
4. **Routing.** A `playInternally` stream is resolved through `YouTubeStreamResolver` (reused; the `youtube_resolving_stream` indicator and `youtube_resolution_failed` message apply) and then played through the existing `routePlayback` internal path, bypassing the player preference. `saveLastLink = false`, as for `ytId` streams.
5. **1080p preference.** `YouTubeStreamResolver` and `InAppYouTubeExtractor.extractSingleUrl` gain an optional maximum height: candidates at or below the cap are kept; if none fit, the lowest available are used. Only tab entries pass the cap; trailers and `ytId` streams do not.
6. **Keep flavor gating on `AppFeaturePolicy`** rather than runtime flavor checks, per the repo convention (`FEATURE_*` flags).

## Risks / Trade-offs

- [Direct results appear in both the original group and the tab] -> intentional so the old list is unchanged; revisit if it confuses users.
- [Copying direct results by rule (any add-on with a YouTube `externalUrl` stream) may catch other add-ons] -> limited to add-ons that also return a YouTube video link.
- [Extractor breakage as YouTube changes] -> failure surfaces `youtube_resolution_failed`; the user can pick another source.
- [Single muxed URL caps quality] -> accepted; the 1080p preference applies within what the muxed/HLS sources offer.
- [Resolution latency delays playback] -> show the existing `youtube_resolving_stream` indicator.
- [False positives for arbitrary `youtube.com` URLs, e.g. channel pages] -> only watch, shorts, embed, live and `youtu.be` forms with a valid id are treated as videos.
- [TV focus handling in `AddonFilterChips`] -> verify the extra chip is reachable with the D-pad.

## Open Questions

- Should the tab's copy of `External Player` keep that name, or be relabelled?
- Should direct `YT-DLP Player` results be moved out of the original group (list changes) instead of copied?
- Should the internal/external/ask preference ever apply to the tab's entries?
