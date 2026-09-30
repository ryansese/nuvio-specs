## 1. Recognition (commonMain)

- [ ] 1.1 Add `isNuvioPlayer`, `youtubeVideoId` and `youtubeWatchUrl` to `StreamItem` (`features/streams/StreamModels.kt`); make `shouldOpenExternally` false for Nuvio Player and leave it unchanged for everything else; leave `StreamParser` and `MetaDetailsParser` untouched
- [ ] 1.2 Add model tests (name match, other name with same URL, non-YouTube URL, `url` present, channel page URL)

## 2. Resolution and routing

- [ ] 2.1 Add an optional `maxHeight` to `TrailerPlaybackResolver.resolveFromYouTubeUrl` (expect, `fullCommonMain` actual, store stubs) and to `InAppYouTubeExtractor`; include the cap in the resolver cache key
- [ ] 2.2 In `openSelectedStream` (`StreamDestination.kt`), resolve Nuvio Player streams into a `PlayerLaunch` with `maxHeight = 1080` before the `shouldOpenExternally` branch and always open the internal player
- [ ] 2.3 Show a resolving indicator and an error message on failure
- [ ] 2.4 Skip writing the resolved URL to `StreamLinkCacheRepository`, and keep `StreamAutoPlaySelector` from selecting Nuvio Player

## 3. Player

- [ ] 3.1 Add `useYoutubeChunkedPlayback` to `PlayerLaunch` and `PlayerScreenArgs` and forward it to `PlatformPlayerSurface`
- [ ] 3.2 Verify the iOS `full` build has a `TrailerPlaybackResolver` actual; add one if missing

## 4. Verification

- [ ] 4.1 Run the project unit tests
- [ ] 4.2 Run a local YouTubio add-on and confirm `External Player` still opens the browser while `Nuvio Player` plays in-app on Android and iOS
- [ ] 4.3 Confirm 1080p is chosen when available and a lower resolution otherwise
- [ ] 4.4 Confirm Play Store and App Store variants keep the previous behavior
