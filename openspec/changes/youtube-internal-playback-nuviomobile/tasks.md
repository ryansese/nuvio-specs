## 1. Parsing and model

- [ ] 1.1 Add `ytId` to `StreamItem` and an `isYouTube` helper (true for `ytId` or YouTube `externalUrl`) in `features/streams/StreamModels.kt`
- [ ] 1.2 Read `ytId` and `yt_id:` ids in `StreamParser.parse` and relax the drop check at `:33`
- [ ] 1.3 Update playable-source checks (`StreamModels.kt` ~`:113`, `:176`) and card visibility (`StreamsScreen.kt` ~`:874`)
- [ ] 1.4 Make `shouldOpenExternally` false for YouTube streams when in-app resolution is available
- [ ] 1.5 Add parser and model tests for `ytId`-only, YouTube `externalUrl` and other `externalUrl` streams

## 2. Resolution and routing

- [ ] 2.1 Add one shared helper that resolves a YouTube stream via `TrailerPlaybackResolver.resolveFromYouTubeUrl` and produces a `PlayerLaunch`
- [ ] 2.2 Call it in `openSelectedStream` (`StreamDestination.kt`) before the `shouldOpenExternally` branch
- [ ] 2.3 Call it in both autoplay paths (~`:305-335` and ~`:470-495`)
- [ ] 2.4 Show a resolving indicator and an error message on failure; keep the browser fallback for stub (store) builds
- [ ] 2.5 Skip writing the resolved URL to `StreamLinkCacheRepository`

## 3. Player

- [ ] 3.1 Add `useYoutubeChunkedPlayback` to `PlayerLaunch` and `PlayerScreenArgs` and forward it to `PlatformPlayerSurface`
- [ ] 3.2 Apply the external-player rule: muxed/HLS resolved URL may go to the external player; separate audio uses the internal player
- [ ] 3.3 Verify the iOS `full` build has a `TrailerPlaybackResolver` actual; add one if missing

## 4. Verification

- [ ] 4.1 Run the project unit tests
- [ ] 4.2 Run a local YouTubio add-on and confirm a result plays internally on Android and on iOS
- [ ] 4.3 Confirm the external-player setting and the per-stream internal/external action behave per the spec
- [ ] 4.4 Confirm Play Store and App Store variants keep the previous behavior without errors
