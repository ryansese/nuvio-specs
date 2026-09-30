## Why

NuvioMobile opens YouTube `externalUrl` streams in the browser, so YouTube results from an add-on such as YouTubio cannot be played inside the app, even though the app already ships an on-device YouTube extractor and a player that supports separate audio, used today only for trailers.

The existing stream lists must not change. In-app playback is therefore offered as an additional **Internal Player** tab.

## What Changes

- Add an **Internal Player** tab to the streams screen. It appears only when the stream list contains YouTube content and lists:
  - copies of the direct-URL results of a group that also carries a YouTube `externalUrl` stream (for YouTubio, the `YT-DLP Player <resolution>` entries), and
  - one entry per YouTube `externalUrl` video, resolved in-app when selected.
- Leave every existing group, card, name and action unchanged: YouTube `externalUrl` entries still open the browser from the original list, `ytId`-only entries stay unlisted, and autoplay is unaffected.
- Resolve entries selected in the Internal Player tab with the in-app extractor and start the internal player with the resolved video (and audio) URLs, preferring 1080p and falling back to lower resolutions. The cap applies to stream playback only, not trailers.
- Tab entries always use the internal player; the external-player setting does not apply to them.
- Pass a "YouTube chunked playback" flag through the player launch so Android avoids googlevideo throttling.
- Do not persist resolved (short-lived) URLs as the "last used link".
- Store-distributed builds (Play Store / App Store), where the extractor is stubbed out, show no Internal Player tab and keep the current behavior.

## Capabilities

### New Capabilities
- `youtube-stream-playback-nuviomobile`: how NuvioMobile (Android and iOS) lists YouTube streams in an Internal Player tab and resolves and plays them in the internal player, including resolution preference, link caching and build-variant gating.

### Modified Capabilities

## Impact

- `NuvioMobile/composeApp/src/commonMain/.../features/streams/` (`StreamModels.kt`, `StreamsScreen.kt`, `StreamsTabletLayout.kt`, `ProviderFilterRow.kt`): derived Internal Player group and a per-stream "play internally" flag
- `commonMain/.../StreamDestination.kt` (`openSelectedStream`)
- `commonMain/.../features/player/` (`PlayerModels.kt`, `PlayerScreenArgs.kt`, `PlayerEngine.kt`)
- `commonMain/.../trailer/TrailerPlaybackResolver.kt` and `fullCommonMain` actuals plus the store stubs (optional `maxHeight`); `androidFull` `YoutubeChunkedDataSourceFactory` (reused)
- iOS libmpv bridge (`iosApp/iosApp/Player/MPVPlayerBridge.swift`): no change expected, verify
- Sibling changes: `youtube-internal-playback-nuviotv`, `youtube-internal-playback-nuviodesktop`
