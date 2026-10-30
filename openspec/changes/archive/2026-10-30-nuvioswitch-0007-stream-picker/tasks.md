## 1. Stream model and rules

- [x] 1.1 Add `src/streams/stream.{hpp,cpp}`: Stream, behaviorHints (proxyHeaders, bingeGroup, filename, videoHash, videoSize) and subtitle parsing from add-on JSON and from inline meta streams
- [x] 1.2 Add `src/streams/stream_rules.{hpp,cpp}`: `supportsStreamResource`, `contentIdFromVideoId`, `dedupKey`, `classify`, `requestHeaders` (removes `Range`), `formatSize`, display name and description fallbacks
- [x] 1.3 Use or add the shared URL segment encoder (Java `URLEncoder` behaviour, `+` → `%20`) and the stream URL builder with base query after `.json`
- [x] 1.4 Add `src/streams/playback_request.hpp`
- [x] 1.5 Unit tests for every rule above, including the content ID examples, de-duplication priority, magnet and `.torrent` detection, and the episode stream URL

## 2. Repository

- [x] 2.1 Add `StreamRepository::fetchAll` with one parallel request per selected add-on, inline fallback on empty, de-duplication within each group, results through `brls::sync`, a generation token and a failed-add-ons list
- [x] 2.2 Order groups by installed add-on order regardless of arrival

## 3. Picker screen

- [x] 3.1 Add `resources/xml/activity/streams.xml`, `resources/xml/views/stream_cell.xml` and `src/activity/stream_activity.{hpp,cpp}` (backdrop, left info column, chips, stream list)
- [x] 3.2 Insert groups as they arrive without moving focus; build chips per add-on with results
- [x] 3.3 Mark non-playable streams ("Needs debrid", "Unsupported") and show a message when selected
- [x] 3.4 Loading, still-loading, no-add-on, and all-failed (with add-on names and Retry) states
- [x] 3.5 Wire Play and episode selection in `DetailsActivity` to push `StreamActivity`; selecting a playable stream builds a `PlaybackRequest` and opens the player (no-op until 0008)
- [x] 3.6 Add the i18n strings

## 4. Verification

- [x] 4.1 `ctest --test-dir build_desktop` passes
- [x] 4.2 Desktop: with Cinemeta and at least one HTTP stream add-on and one torrent add-on installed, open a movie and an episode; check order, chips, de-duplication, "Needs debrid" labels, and that a slow add-on doesn't block others
- [x] 4.3 Desktop: check that the `PlaybackRequest` logged for a stream with `proxyHeaders` has no `Range` header and includes its subtitles
- [x] 4.4 Switch: build the `.nro` and check the picker on hardware or in an emulator with a large result list (scrolling, focus stability while groups arrive)
