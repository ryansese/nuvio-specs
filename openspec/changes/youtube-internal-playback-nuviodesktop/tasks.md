> Commit `6c4aba22` on NuvioDesktop `Dev` holds an earlier approach (parses `ytId`, plays any YouTube `externalUrl` stream internally, changes autoplay, sets a non-existent mpv `audio-file` option, no locale fix). It conflicts with the current design and is to be reworked by the tasks below; only 3.1 still applies.

## 1. Recognition (commonMain)

- [ ] 1.1 Rework `StreamItem`: add `isNuvioPlayer`, `youtubeVideoId` and `youtubeWatchUrl`; drop `ytId` and any name-agnostic YouTube handling; keep `shouldOpenExternally` unchanged except false for Nuvio Player
- [ ] 1.2 Revert `StreamParser` to its original behavior (no `ytId`, original drop check) and remove the autoplay change in `StreamAutoPlaySelector`; make sure autoplay never selects Nuvio Player
- [ ] 1.3 Add model tests (name match, other name with same URL, non-YouTube URL, `url` present, channel page URL) and parser tests confirming `ytId`-only streams stay dropped

## 2. Resolution and routing

- [ ] 2.1 Add an optional `maxHeight` to `TrailerPlaybackResolver.resolveFromYouTubeUrl` (expect and actuals) and to `InAppYouTubeExtractor` (progressive, adaptive video and HLS candidates; fall back to the lowest available); include the cap in the resolver cache key
- [ ] 2.2 In `openSelectedStream` (`StreamDestination.kt`), resolve Nuvio Player streams into a `PlayerLaunch` with `maxHeight = 1080` before the `shouldOpenExternally` branch
- [ ] 2.3 Show an error message on failure
- [ ] 2.4 Skip writing the resolved URL to `StreamLinkCacheRepository`
- [ ] 2.5 Confirm the external-player branch is never taken for Nuvio Player on desktop, including Windows

## 3. Desktop player

- [x] 3.1 Forward `sourceAudioUrl` from `PlayerEngine.desktop.kt` into `NativePlayerController.attach` and add an audio parameter to `NativePlayerBridge.create`
- [ ] 3.2 Set `audio-files` as a one-item node array before `mpv_initialize` in `native/macos/player_bridge.mm`, `native/linux/player_bridge.cpp` and `native/windows/player_bridge.cpp` (replace the string `audio-file` option)
- [ ] 3.3 Call `setlocale(LC_NUMERIC, "C")` before `mpv_create` in the macOS bridge (and Windows if it fails there)
- [ ] 3.4 Measure playback of a long video and add HLS preference, range-chunked requests or mpv options to avoid throttling stalls

## 4. Verification

- [ ] 4.1 Run the project unit tests (the known failures are `NativePlayerControllerTeardownTest`, which cannot load libmpv from `@rpath`, and 4 UI tests)
- [ ] 4.2 Run a local YouTubio add-on and confirm `External Player` still opens the system browser while `Nuvio Player` plays with video and audio on macOS, Windows and Linux
- [ ] 4.3 Confirm 1080p is chosen when available and a lower resolution otherwise
- [ ] 4.4 Confirm a video longer than 10 minutes plays without repeated stalls
