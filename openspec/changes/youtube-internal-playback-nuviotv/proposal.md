## Why

NuvioTV already resolves YouTube streams that carry a `ytId` and plays them in the internal player. Streams whose only pointer is a `youtube.com` / `youtu.be` `externalUrl` (the shape a YouTubio-style add-on returns) are treated as external and opened in a browser, which is unusable on a TV.

The existing stream list must not change, including how `ytId` streams behave today. In-app playback of the `externalUrl` shape is therefore offered as an additional **Internal Player** tab.

## What Changes

- Add an **Internal Player** tab to the streams screen (an extra entry in the add-on filter chips). It appears only when the stream list contains a YouTube `externalUrl` stream and lists:
  - copies of the direct-URL results of the add-on that also returned that stream (for YouTubio, the `YT-DLP Player <resolution>` entries), and
  - one entry per YouTube `externalUrl` video, resolved on-device when selected.
- Leave every existing add-on group, card, name and action unchanged: YouTube `externalUrl` cards still open the browser from the original list, `ytId` streams keep their current resolve-and-play behavior, and autoplay and in-player source switching are unaffected.
- Resolve entries selected in the Internal Player tab with the existing on-device resolver and play them in the internal player, preferring 1080p and falling back to lower resolutions. The cap applies to stream playback only, not trailers.
- Tab entries always use the internal player; the internal/external/ask preference does not apply to them.
- Show no Internal Player tab in the `playstore` flavor, where in-app YouTube resolution is disabled; behavior there is unchanged.

## Capabilities

### New Capabilities
- `youtube-stream-playback-nuviotv`: how NuvioTV lists YouTube `externalUrl` streams in an Internal Player tab and resolves and plays them in the internal player, including resolution preference and flavor gating.

### Modified Capabilities

## Impact

- `NuvioTV/app/src/main/java/com/nuvio/tv/domain/model/Stream.kt` (YouTube URL detection, per-stream "play internally" flag; `isExternal()` unchanged for originals)
- `ui/screens/stream/StreamScreenUiState.kt`, `StreamScreenViewModel.kt`, `StreamScreen.kt` (derived tab entry in the add-on filter chips, routing of tab entries)
- `core/streams/YouTubeStreamResolver.kt`, `data/trailer/InAppYouTubeExtractor.kt` (reused; optional resolution cap)
- Tests: `StreamYouTubeTest`, `YouTubeStreamResolverTest`
- Sibling changes: `youtube-internal-playback-nuviomobile`, `youtube-internal-playback-nuviodesktop`
