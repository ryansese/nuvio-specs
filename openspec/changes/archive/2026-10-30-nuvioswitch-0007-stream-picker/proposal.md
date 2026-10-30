# NuvioSwitch-0007 Stream picker

## Why

The details screen (0006) produces a video ID, but nothing yet turns that into a playable link. In NuvioTV, the stream picker asks every add-on that serves streams for that ID. It shows results grouped by add-on as they arrive, and passes the chosen stream with its headers and subtitles to the player. Playback (0008) needs this as its input.

## What Changes

- Add a stream picker screen, opened from Play or an episode on the details screen.
  - Layout, following `ui/screens/stream/StreamScreen.kt`:
    - a dimmed backdrop;
    - left column: logo or title, plus the episode line;
    - right column: add-on filter chips ("All" plus one per add-on that returned streams) above a list of stream cards.
  - Each card shows the stream's `name` (often the quality) and `title`/`description` lines, the add-on name, and the file size from `behaviorHints.videoSize` when present.
- **Which add-ons are queried** (`core/streams/PlaybackAvailability.kt`, `supportsStreamResource`):
  - enabled add-ons with a `stream` resource whose types are empty or include the type;
  - the video ID must match the resource's `idPrefixes`, or the add-on's, or there are none;
  - all are queried in parallel at `{base}/stream/{type}/{videoId}.json` (`data/repository/StreamRepositoryImpl.kt`).
- **Inline fallback:** when an add-on returns no streams, its meta is fetched with the content-level ID, and the streams listed inline for the matching video are used (`fetchInlineStreamsFromMeta`).
- **Grouping:** results are grouped per add-on and ordered by installed add-on order, whatever order they arrive in (`core/player/StreamAutoPlaySelector.kt`, `orderAddonStreams`). Duplicates within a group are dropped by `infoHash:fileIdx`, then URL (+ name), then `externalUrl`, then `ytId` (`StreamRepositoryImpl.dedupKey`).
- **Playable vs not** (`domain/model/Stream.kt`):
  - HTTP(S) streams can be played;
  - torrent-only streams (`infoHash` or `magnet:`/`.torrent` with no HTTP URL) are shown dimmed and labelled as needing debrid (0018);
  - `externalUrl`-only streams and YouTube `ytId`-only streams are shown as unsupported.
- Selecting a playable stream opens the player (0008). The hand-off carries:
  - the URL;
  - the request headers from `behaviorHints.proxyHeaders.request` minus `Range`;
  - the stream's own subtitles;
  - `bingeGroup`, `filename`, `videoHash` and `videoSize`;
  - the content details from 0006.
- **Loading and errors:**
  - a loading state until the first group arrives;
  - an error with Retry when every add-on fails or returns nothing;
  - a "No add-on provides streams for this title" message when no add-on qualifies.

## Capabilities

### New Capabilities
- `nuvioswitch-stream-picker`: which add-ons are asked for streams, how responses are merged, ordered and de-duplicated, playability rules, the picker UI, and the data handed to the player.

### Modified Capabilities
None.

## Impact

- **Code (NuvioSwitch):**
  - `src/streams/` holds the Stream model and parser (including behaviorHints and subtitles), the stream repository (add-on selection, parallel fetch, inline fallback, de-duplication, ordering) and pure playability helpers;
  - `src/activity/stream_activity.{hpp,cpp}`;
  - `resources/xml/activity/streams.xml` and `resources/xml/views/stream_cell.xml`;
  - i18n strings.
- **Reuses from earlier changes:** the add-on model and order (0003); the meta model and its inline `videos[].streams` raw JSON, plus the meta URL builder (0006); HTTP and the image loader (0002).
- **Depends on:** 0002, 0003, 0006. Selecting a stream hands off to 0008.
- **Out of scope:**
  - auto-play and binge-group reuse (0010);
  - debrid resolution (0018) and local torrent playback (never on Switch);
  - QuickJS plugin scrapers, YouTube resolution and external players;
  - the "last link" cache (`StreamLinkCacheDataStore`).
