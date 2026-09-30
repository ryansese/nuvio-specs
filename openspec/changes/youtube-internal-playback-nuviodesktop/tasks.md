## 1. Parsing and model (shared commonMain, mirrors NuvioMobile)

- [x] 1.1 Add `ytId` to `StreamItem` and an `isYouTube` helper in `features/streams/StreamModels.kt`
- [x] 1.2 Read `ytId` and `yt_id:` ids in `StreamParser.parse` and relax the drop check at `:33`
- [x] 1.3 Update playable-source checks and stream-card visibility so YouTube streams are listed
- [x] 1.4 Make `shouldOpenExternally` false for YouTube streams
- [x] 1.5 Add parser and model tests

## 2. Resolution and routing

- [x] 2.1 Add a shared helper resolving via `TrailerPlaybackResolver.resolveFromYouTubeUrl` into a `PlayerLaunch`
- [x] 2.2 Call it in `openSelectedStream` (`StreamDestination.kt`) before the `shouldOpenExternally` branch and in both autoplay paths
- [x] 2.3 Show an error message on failure
- [x] 2.4 Skip writing the resolved URL to `StreamLinkCacheRepository`
- [x] 2.5 Confirm the external-player branch is never taken for YouTube on desktop, including Windows

## 3. Desktop player

- [x] 3.1 Forward `sourceAudioUrl` from `PlayerEngine.desktop.kt` into `NativePlayerController.attach`
- [x] 3.2 Add an audio parameter to `NativePlayerBridge.create` and implement mpv `audio-file` in `native/macos/player_bridge.mm`, `native/linux/player_bridge.cpp`, `native/windows/player_bridge.cpp`
- [x] 3.3 If a bridge cannot be changed, request a muxed or HLS source on that platform instead (not needed: all three bridges take an audio URL)
- [ ] 3.4 Measure playback of a long video and add mpv options or HLS preference to avoid throttling stalls

## 4. Verification

- [ ] 4.1 Run the project unit tests
- [ ] 4.2 Run a local YouTubio add-on and confirm a result plays with video and audio on macOS, Windows and Linux
- [ ] 4.3 Confirm a video longer than 10 minutes plays without repeated stalls
