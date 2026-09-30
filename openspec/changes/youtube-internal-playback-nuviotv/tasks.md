## 1. Model and Internal Player tab

- [ ] 1.1 Add a YouTube `externalUrl` video-id helper and a `playInternally` flag to `Stream` (`domain/model/Stream.kt`) using `InAppYouTubeExtractor.extractVideoId` for watch, shorts, embed, live and `youtu.be` forms only; leave `isExternal()` and `youTubeIdToResolve()` unchanged for originals
- [ ] 1.2 Derive the `Internal Player` entry for the add-on filter chips in the stream screen UI state/ViewModel (hidden when in-app resolution is disabled); keep `selectedAddonFilter` working for real add-ons
- [ ] 1.3 Add cases to `StreamYouTubeTest` and a state test for the tab (copies, dedupe, absent without YouTube content, originals untouched, non-YouTube `externalUrl`, torrent and direct-debrid streams)

## 2. Routing and resolution

- [ ] 2.1 Route `playInternally` streams through `YouTubeStreamResolver` and the existing internal `routePlayback` path, ignoring the player preference, with `saveLastLink = false`
- [ ] 2.2 Add an optional maximum height to `YouTubeStreamResolver` and `InAppYouTubeExtractor.extractSingleUrl` (cap, fall back to lowest available) and pass 1080 only for tab entries
- [ ] 2.3 Confirm the resolving indicator and the failure message show and that the browser is not opened on failure
- [ ] 2.4 Confirm the original cards, autoplay (`StreamAutoPlaySelector`) and in-player switching (`PlayerRuntimeControllerStreams.kt`) are untouched

## 3. Flavors

- [ ] 3.1 Confirm `full` shows the tab and resolves in-app and `playstore` shows no tab and keeps the watch-page `externalUrl` fallback (`YouTubeStreamResolverTest`)

## 4. Verification

- [ ] 4.1 Run `./gradlew :app:testFullDebugUnitTest`
- [ ] 4.2 Run a local YouTubio add-on and confirm the original list is unchanged and the Internal Player tab plays a result in the internal player on the emulator (add-on host `10.0.2.2`)
- [ ] 4.3 Confirm the extra chip is reachable with the D-pad
- [ ] 4.4 Confirm 1080p is chosen when available and a lower resolution otherwise
