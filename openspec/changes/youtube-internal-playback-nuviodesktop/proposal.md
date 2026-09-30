## Why

NuvioDesktop opens YouTube `externalUrl` streams in the system browser, and its player cannot play YouTube add-on results that need the in-app extractor or carry separate video and audio streams. The app already ships an on-device YouTube extractor (used for trailers) and a libmpv player, so these videos could play in-app.

The existing stream lists must not change: users rely on them, and add-ons such as YouTubio label entries in ways the app should not rewrite. The internal playback is therefore offered as an additional **Internal Player** tab.

## What Changes

- Add an **Internal Player** tab to the streams screen. It appears only when the stream list contains YouTube content and it lists:
  - the direct-URL results of a group that also carries a YouTube `externalUrl` stream (for YouTubio these are the `YT-DLP Player <resolution>` entries), and
  - one entry per YouTube `externalUrl` video, resolved in-app when selected.
- Leave every existing group, card, name and action unchanged: YouTube `externalUrl` entries such as `External Player` still open the system browser from the original list, `ytId`-only entries (`Stremio Player`) stay unlisted, and autoplay is unaffected.
- Resolve entries selected in the Internal Player tab with the in-app extractor and play them in the internal libmpv player, preferring 1080p and falling back to lower resolutions. The cap applies to stream playback only, not trailers.
- Forward the resolved audio URL through the desktop player stack to mpv (`audio-files`), so separate video and audio sources play with sound.
- Make libmpv start on macOS by setting `LC_NUMERIC` to `C` before `mpv_create`.
- Keep stream playback internal on Windows even though the trailer playback mode there is external.
- Do not persist resolved URLs as the "last used link".

## Capabilities

### New Capabilities
- `youtube-stream-playback-nuviodesktop`: how NuvioDesktop lists YouTube streams in an Internal Player tab and resolves and plays them in the internal libmpv player on macOS, Windows and Linux, including resolution preference, audio-URL forwarding and throttling considerations.

### Modified Capabilities

## Impact

- `NuvioDesktop/composeApp/src/commonMain/.../features/streams/` (`StreamModels.kt`, `StreamsScreen.kt`, `StreamsTabletLayout.kt`): derived Internal Player group and a per-stream "play internally" flag
- `commonMain/.../StreamDestination.kt`: in-app resolution for Internal Player entries
- `fullCommonMain/.../trailer/InAppYouTubeExtractor.kt`, `TrailerPlaybackResolver.kt` (plus the `iosAppStore` and `androidPlaystore` actuals): optional `maxHeight`
- `desktopMain/.../features/player/PlayerEngine.desktop.kt`, `desktop/NativePlayerController.kt`, `desktop/NativePlayerBridge.kt`
- `desktopMain/native/{macos/player_bridge.mm, linux/player_bridge.cpp, windows/player_bridge.cpp}`: `audio-files` option; macOS locale call
- `desktopMain/.../core/build/AppFeaturePolicy.desktop.kt` (read only)
- Sibling changes: `youtube-internal-playback-nuviotv`, `youtube-internal-playback-nuviomobile`
