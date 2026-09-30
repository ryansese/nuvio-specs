## Why

In NuvioMobile a stream that carries only a `ytId` is silently dropped by `StreamParser`, and a stream whose `externalUrl` is a YouTube link is opened in the browser. YouTube results (for example from a YouTubio add-on) therefore cannot be played inside the app, even though the app already ships an on-device YouTube extractor and a player that supports separate audio, used today only for trailers.

## What Changes

- Parse `ytId` (and `yt_id:`-prefixed ids) on streams so they are no longer dropped, and recognise YouTube `externalUrl` streams.
- When a YouTube stream is selected (manually or via autoplay), resolve it with the in-app extractor and start the internal player with the resolved video (and audio) URLs.
- Obey the existing external-player setting; external launch needs a directly playable muxed/HLS URL, otherwise fall back to the internal player.
- Pass a "YouTube chunked playback" flag through the player launch so Android avoids googlevideo throttling.
- Do not persist resolved (short-lived) URLs as the "last used link".
- Store-distributed builds (Play Store / App Store), where the extractor is stubbed out, keep the current behavior with a clear failure message.

## Capabilities

### New Capabilities
- `youtube-stream-playback-nuviomobile`: how NuvioMobile (Android and iOS) parses, resolves and plays YouTube streams in the internal player, including preference handling, autoplay, link caching and build-variant gating.

### Modified Capabilities

## Impact

- `NuvioMobile/composeApp/src/commonMain/.../features/streams/` (`StreamModels.kt`, `StreamParser.kt`, `StreamsScreen.kt`)
- `commonMain/.../StreamDestination.kt` (`openSelectedStream` and the two autoplay paths)
- `commonMain/.../features/player/` (`PlayerModels.kt`, `PlayerScreenArgs.kt`, `PlayerEngine.kt`, `ExternalPlayerPlatform.kt`)
- `commonMain/.../trailer/TrailerPlaybackResolver.kt` and `fullCommonMain` actuals (reused); `androidFull` `YoutubeChunkedDataSourceFactory` (reused)
- iOS libmpv bridge (`iosApp/iosApp/Player/MPVPlayerBridge.swift`): no change expected, verify
- Sibling changes: `youtube-internal-playback-nuviotv`, `youtube-internal-playback-nuviodesktop`
