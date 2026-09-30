## 1. Model and Internal Player tab (commonMain)

- [ ] 1.1 Add `playInternally`, `youtubeWatchUrl`, `youtubeVideoId`, `isInternalYouTubeCandidate` and `isYouTube` to `StreamItem` (`features/streams/StreamModels.kt`); keep `shouldOpenExternally` unchanged for originals and leave `StreamParser` untouched
- [ ] 1.2 Add `StreamsUiState.displayGroups` with the derived `Internal Player` group (hidden when the resolver is a stub); use it in `filteredGroups` and in the `ProviderFilterRow` call sites in `StreamsScreen.kt` and `StreamsTabletLayout.kt`
- [ ] 1.3 Add model tests for the group (copies, dedupe, no group without YouTube content, originals untouched)

## 2. Resolution and routing

- [ ] 2.1 Add an optional `maxHeight` to `TrailerPlaybackResolver.resolveFromYouTubeUrl` (expect, `fullCommonMain` actual, store stubs) and to `InAppYouTubeExtractor`; include the cap in the resolver cache key
- [ ] 2.2 In `openSelectedStream` (`StreamDestination.kt`), resolve `playInternally` streams into a `PlayerLaunch` with `maxHeight = 1080` before the `shouldOpenExternally` branch and always open the internal player
- [ ] 2.3 Show a resolving indicator and an error message on failure
- [ ] 2.4 Skip writing the resolved URL to `StreamLinkCacheRepository`

## 3. Player

- [ ] 3.1 Add `useYoutubeChunkedPlayback` to `PlayerLaunch` and `PlayerScreenArgs` and forward it to `PlatformPlayerSurface`
- [ ] 3.2 Verify the iOS `full` build has a `TrailerPlaybackResolver` actual; add one if missing

## 4. Verification

- [ ] 4.1 Run the project unit tests
- [ ] 4.2 Run a local YouTubio add-on and confirm the original list is unchanged and the Internal Player tab plays a result on Android and on iOS
- [ ] 4.3 Confirm 1080p is chosen when available and a lower resolution otherwise
- [ ] 4.4 Confirm Play Store and App Store variants show no tab and keep the previous behavior
