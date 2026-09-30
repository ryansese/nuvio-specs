## Why

NuvioTV already resolves YouTube streams that carry a `ytId` and plays them in the internal player. Streams whose only pointer is a `youtube.com` / `youtu.be` `externalUrl` (the shape a YouTubio-style add-on can return) are treated as external and opened in a browser, which is unusable on a TV. YouTube videos should play in-app regardless of which of the two shapes the add-on uses.

## What Changes

- Treat a stream whose `externalUrl` is a YouTube watch/short/embed URL as a resolvable YouTube stream, using the same on-device resolution path as `ytId` streams.
- Stop the browser fallback (`openExternalInBrowser` and the in-player equivalent) from claiming those streams while in-app resolution is available.
- Allow such streams to participate in autoplay selection and in-player source/episode switching.
- Keep obeying the existing player preference (internal / external / ask-every-time); external launch receives the resolved URL.
- Keep the current behavior in the `playstore` flavor, where in-app YouTube resolution is disabled.

## Capabilities

### New Capabilities
- `youtube-stream-playback-nuviotv`: how NuvioTV detects, resolves and plays YouTube streams (`ytId` and YouTube `externalUrl`) in the internal player, including preference handling, autoplay, and flavor gating.

### Modified Capabilities

## Impact

- `NuvioTV/app/src/main/java/com/nuvio/tv/domain/model/Stream.kt` (`youTubeIdToResolve()`, `isExternal()`)
- `core/streams/YouTubeStreamResolver.kt`, `data/trailer/InAppYouTubeExtractor.kt` (reused, `extractVideoId`)
- `ui/screens/stream/StreamScreen.kt`, `StreamScreenViewModel.kt`, `StreamAutoPlaySelector.kt`
- `ui/screens/player/PlayerRuntimeControllerStreams.kt`
- Tests: `StreamYouTubeTest`, `YouTubeStreamResolverTest`
- Sibling changes: `youtube-internal-playback-nuviomobile`, `youtube-internal-playback-nuviodesktop`
