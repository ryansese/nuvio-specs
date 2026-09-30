## 1. Detection

- [ ] 1.1 Extend `Stream.youTubeIdToResolve()` (`domain/model/Stream.kt`) to derive the id from a YouTube `externalUrl` using `InAppYouTubeExtractor.extractVideoId`, for watch, shorts, embed and `youtu.be` forms only
- [ ] 1.2 Make `Stream.isExternal()` return false for streams detected in 1.1
- [ ] 1.3 Add cases to `StreamYouTubeTest` for `ytId`, YouTube `externalUrl`, non-YouTube `externalUrl`, torrent and direct-debrid streams

## 2. Routing

- [ ] 2.1 Confirm `openExternalInBrowser` in `StreamScreen.kt` no longer claims detected streams; adjust if it checks the URL rather than `isExternal()`
- [ ] 2.2 Confirm the same in `PlayerRuntimeControllerStreams.kt` (`openExternalStreamInBrowser`, `switchToSourceStream`, `switchToEpisodeStream`)
- [ ] 2.3 Verify `StreamAutoPlaySelector` now considers detected streams and add a test
- [ ] 2.4 Verify player preference handling (Internal / External / Ask) passes the resolved URL to the external player

## 3. Flavors

- [ ] 3.1 Confirm `full` resolves in-app and `playstore` keeps the watch-page `externalUrl` fallback (`YouTubeStreamResolverTest`)

## 4. Verification

- [ ] 4.1 Run `./gradlew :app:testFullDebugUnitTest`
- [ ] 4.2 Run a local YouTubio add-on and confirm a result plays in the internal player on the emulator (add-on host `10.0.2.2`)
- [ ] 4.3 Confirm failure shows the YouTube resolution failure message and does not open the browser
