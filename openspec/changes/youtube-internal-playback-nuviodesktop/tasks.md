> Commit `6c4aba22` on NuvioDesktop `Dev` holds an earlier approach (parses `ytId`, plays YouTube `externalUrl` streams internally in the original list, changes autoplay, sets a non-existent mpv `audio-file` option, no locale fix). It conflicts with the current design and is to be reworked by the tasks below; only 3.1 still applies.

## 1. Model and Internal Player tab (commonMain)

- [ ] 1.1 Rework `StreamItem`: add `playInternally`, `youtubeWatchUrl`, `youtubeVideoId`, `isInternalYouTubeCandidate` and `isYouTube`; drop `ytId`; keep `shouldOpenExternally` unchanged for originals
- [ ] 1.2 Revert `StreamParser` to its original behavior (no `ytId`, original drop check) and remove the autoplay change in `StreamAutoPlaySelector`
- [ ] 1.3 Add `StreamsUiState.displayGroups` with the derived `Internal Player` group; use it in `filteredGroups` and in the `ProviderFilterRow` call sites in `StreamsScreen.kt` and `StreamsTabletLayout.kt`
- [ ] 1.4 Add model tests for the group (copies, dedupe, no group without YouTube content, originals untouched) and parser tests confirming `ytId`-only streams stay dropped

## 2. Resolution and routing

- [ ] 2.1 Add an optional `maxHeight` to `TrailerPlaybackResolver.resolveFromYouTubeUrl` (expect and actuals) and to `InAppYouTubeExtractor` (progressive, adaptive video and HLS candidates; fall back to the lowest available); include the cap in the resolver cache key
- [ ] 2.2 In `openSelectedStream` (`StreamDestination.kt`), resolve `playInternally` streams into a `PlayerLaunch` with `maxHeight = 1080`, before the `shouldOpenExternally` branch; remove the autoplay branch
- [ ] 2.3 Show an error message on failure
- [ ] 2.4 Skip writing the resolved URL to `StreamLinkCacheRepository`
- [ ] 2.5 Confirm the external-player branch is never taken for the tab's entries on desktop, including Windows

## 3. Desktop player

- [x] 3.1 Forward `sourceAudioUrl` from `PlayerEngine.desktop.kt` into `NativePlayerController.attach` and add an audio parameter to `NativePlayerBridge.create`
- [ ] 3.2 Set `audio-files` as a one-item node array before `mpv_initialize` in `native/macos/player_bridge.mm`, `native/linux/player_bridge.cpp` and `native/windows/player_bridge.cpp` (replace the string `audio-file` option)
- [ ] 3.3 Call `setlocale(LC_NUMERIC, "C")` before `mpv_create` in the macOS bridge (and Windows if it fails there)
- [ ] 3.4 Measure playback of a long video and add HLS preference, range-chunked requests or mpv options to avoid throttling stalls

## 4. Verification

- [ ] 4.1 Run the project unit tests (the known failures are `NativePlayerControllerTeardownTest`, which cannot load libmpv from `@rpath`, and 4 UI tests)
- [ ] 4.2 Run a local YouTubio add-on and confirm the original list is unchanged and the Internal Player tab plays a result with video and audio on macOS, Windows and Linux
- [ ] 4.3 Confirm 1080p is chosen when available and a lower resolution otherwise
- [ ] 4.4 Confirm a video longer than 10 minutes plays without repeated stalls
