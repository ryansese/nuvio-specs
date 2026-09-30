## 1. Recognition

- [ ] 1.1 Add `Stream.isNuvioPlayer()` and make `isExternal()` false for it in `domain/model/Stream.kt` (exact name `Nuvio Player`, no `url`, no torrent/debrid, YouTube-host `externalUrl` (`youtu.be`, `youtube.com`, `*.youtube.com`) with a valid video id from `InAppYouTubeExtractor.extractVideoId`); extend `youTubeIdToResolve()` to return the id for it; leave every other stream's behavior unchanged
- [ ] 1.2 Add cases to `StreamYouTubeTest` (name match, other name with same URL, non-YouTube URL, `url` present, torrent and debrid, channel page URL)

## 2. Routing and resolution

- [ ] 2.1 Route Nuvio Player streams through `resolveStreamForPlayback` / `YouTubeStreamResolver` and the existing `routePlayback` path, ignoring the player preference, with `saveLastLink = false`
- [ ] 2.2 Add an optional maximum height to `YouTubeStreamResolver` and `InAppYouTubeExtractor.extractSingleUrl` (cap, fall back to lowest available) and pass 1080 only for Nuvio Player streams
- [ ] 2.3 Keep `StreamAutoPlaySelector` from selecting Nuvio Player streams
- [ ] 2.4 Confirm the resolving indicator and failure message show and the browser is not opened on failure

## 3. Flavors

- [ ] 3.1 Gate on `AppFeaturePolicy`: `full` plays in-app, `playstore` keeps the watch-page `externalUrl` behavior (`YouTubeStreamResolverTest`)

## 4. Verification

- [ ] 4.1 Run `./gradlew :app:testFullDebugUnitTest`
- [ ] 4.2 Run a local YouTubio add-on and confirm `External Player` still opens the browser while `Nuvio Player` plays in the internal player on the emulator (add-on host `10.0.2.2`)
- [ ] 4.3 Confirm 1080p is chosen when available and a lower resolution otherwise
