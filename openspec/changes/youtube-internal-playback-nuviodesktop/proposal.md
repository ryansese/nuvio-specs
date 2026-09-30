## Why

NuvioDesktop has the same gap as NuvioMobile: `ytId`-only streams are dropped by `StreamParser`, and YouTube `externalUrl` streams are opened in the system browser. Desktop additionally cannot play separate video and audio streams (the audio URL is never forwarded to the libmpv bridge) and has no YouTube throttling mitigation, so simply reusing the trailer extractor would yield silent or throttled playback.

## What Changes

- Parse `ytId` (and `yt_id:`-prefixed ids) on streams and recognise YouTube `externalUrl` streams (shared `commonMain` behavior, same as NuvioMobile).
- Resolve selected YouTube streams with the in-app extractor and play them in the internal libmpv player, for manual selection and autoplay.
- Forward the resolved audio URL through the desktop player stack to mpv so separate video+audio sources play with sound; where not feasible, prefer a muxed or HLS source.
- Keep the external player path out of scope: it is disabled on desktop (`externalPlayerSupported = false`), so YouTube streams always use the internal player.
- Keep stream playback internal on Windows even though the trailer playback mode there is external.
- Do not persist resolved URLs as the "last used link".

## Capabilities

### New Capabilities
- `youtube-stream-playback-nuviodesktop`: how NuvioDesktop parses, resolves and plays YouTube streams in the internal libmpv player on macOS, Windows and Linux, including audio-URL forwarding and throttling considerations.

### Modified Capabilities

## Impact

- `NuvioDesktop/composeApp/src/commonMain/.../features/streams/` and `StreamDestination.kt` (as in NuvioMobile)
- `desktopMain/.../features/player/PlayerEngine.desktop.kt`, `desktop/NativePlayerController.kt`, `desktop/NativePlayerBridge.kt`
- `desktopMain/native/{macos/player_bridge.mm, linux/player_bridge.cpp, windows/player_bridge.cpp}` (mpv audio-file option)
- `fullCommonMain/.../trailer/InAppYouTubeExtractor.kt`, `TrailerPlaybackResolver.kt` (reused; compiled into desktop via `build.gradle.kts`)
- `desktopMain/.../core/build/AppFeaturePolicy.desktop.kt` (read only)
- Sibling changes: `youtube-internal-playback-nuviotv`, `youtube-internal-playback-nuviomobile`
