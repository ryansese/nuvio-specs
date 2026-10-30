## Purpose

TBD: Define NuvioSwitch Stream picker: which add-ons are asked for streams, how responses are merged, ordered and de-duplicated, playability rules, the picker UI, and the data handed to the player.

## Requirements

### Requirement: Stream add-on selection
For a given type and video ID, the app SHALL query every enabled add-on that has a `stream` resource whose `types` is empty or contains the type, and whose ID prefixes (from the `stream` resource if it declares any, otherwise from the add-on) are empty or include a prefix the video ID starts with. All selected add-ons SHALL be queried in parallel.

#### Scenario: Prefix filter
- **WHEN** add-on A's `stream` resource declares prefix `tt` and add-on B's declares `kitsu:`, and the video ID is `tt0903747:2:3`
- **THEN** only A is queried

#### Scenario: No qualifying add-on
- **WHEN** no enabled add-on has a matching `stream` resource
- **THEN** no request is sent and the picker shows "No add-on provides streams for this title"

### Requirement: Stream request URL
Stream requests SHALL use `{basePath}/stream/{type}/{videoId}.json{baseQuery}`, with type and video ID encoded like Java's `URLEncoder` (form encoding, `:` becomes `%3A`) with `+` replaced by `%20`, and any base `?query` appended after `.json`. This matches NuvioTV's `encodePathSegment`.

#### Scenario: Episode request
- **WHEN** the add-on base URL is `https://torrentio.example/cfg?x=1` and the video is `series`/`tt0903747:2:3`
- **THEN** the request URL is `https://torrentio.example/cfg/stream/series/tt0903747%3A2%3A3.json?x=1`

### Requirement: Inline stream fallback
When an add-on's stream response succeeds but contains no streams, the app SHALL fetch that add-on's meta for the content-level ID and use the `streams` of the video whose `id` equals the video ID. The content-level ID SHALL be the video ID with up to two trailing numeric `:`-separated segments removed, keeping at least one segment for IDs starting with `tt` or a number and at least two segments otherwise.

#### Scenario: Content ID derivation
- **WHEN** the video IDs are `tt1234567:1:5`, `mal:63375:1:5` and `kitsu:12345:2`
- **THEN** the meta IDs used are `tt1234567`, `mal:63375` and `kitsu:12345`

#### Scenario: Inline streams used
- **WHEN** an add-on returns `{"streams": []}` and its meta for the content has the video with two inline streams
- **THEN** those two streams appear under that add-on's group

### Requirement: Grouping, order and de-duplication
Streams SHALL be shown grouped by add-on, keeping each add-on's own stream order. Groups SHALL be ordered by installed add-on order regardless of response arrival order, and each group SHALL appear as soon as its response arrives. Within a group, streams with the same key SHALL be shown once, where the key is the lowercased `infoHash` plus `fileIdx` when an info hash is present, otherwise the `url` plus its `name`, otherwise `externalUrl`, otherwise `ytId`, otherwise add-on name, name and title.

#### Scenario: Out-of-order arrival
- **WHEN** add-ons are installed as A, B, C and C answers first, then A
- **THEN** the list shows C's group first, then A's group is inserted above it when it arrives, and the final order is A, then C

#### Scenario: Duplicate removed
- **WHEN** an add-on returns two streams with the same info hash and file index
- **THEN** only the first is shown

### Requirement: Add-on filter chips
The picker SHALL show an "All" chip plus one chip per add-on that returned at least one stream, in group order. Selecting a chip SHALL show only that add-on's streams; "All" SHALL show every group.

#### Scenario: Filter by add-on
- **WHEN** the user selects the chip for add-on B
- **THEN** only B's streams are listed and focus moves to the first of them

### Requirement: Stream cards
Each stream SHALL be shown as a card with its `name` (or `title`, or `description`, or "Unknown stream" as fallback), its description (`description`, else `title`) on up to three lines, the add-on name, and the size from `behaviorHints.videoSize` formatted in GB or MB when present.

#### Scenario: Card text
- **WHEN** a stream has `name` "Torrentio\n4K" and `title` "Movie.2160p.mkv\n💾 12.3 GB"
- **THEN** the card shows "Torrentio 4K" as its heading and the title lines below it

### Requirement: Playability
A stream SHALL be playable when its `url` is an HTTP or HTTPS URL that is not a `.torrent` URL (an `externalUrl` is never played). Torrent-only streams (an info hash, magnet link, `torrent:` link or `.torrent` URL and no playable `url`) SHALL be shown dimmed with a "Needs debrid" label. Streams with only an `externalUrl` web link, or only a `ytId`, SHALL be shown dimmed with an "Unsupported" label. Selecting a non-playable stream SHALL show a short explanation and SHALL NOT open the player.

#### Scenario: HTTP stream
- **WHEN** a stream has `url` `https://cdn.example/movie.mp4`
- **THEN** it is playable

#### Scenario: Torrent only
- **WHEN** a stream has `infoHash` and `fileIdx` but no `url`
- **THEN** it is shown with "Needs debrid" and selecting it shows a message instead of playing

### Requirement: Player hand-off
Selecting a playable stream SHALL open the player with: the stream URL; request headers from `behaviorHints.proxyHeaders.request` with any `Range` header removed (case-insensitive); the stream's own `subtitles` (URL, language, ID); `behaviorHints.bingeGroup`, `filename`, `videoHash` and `videoSize`; the add-on name; and the content type, content ID, video ID, season, episode and display data received from the details screen.

#### Scenario: Headers passed without Range
- **WHEN** a stream has `proxyHeaders.request` `{"Referer": "https://a.io", "range": "bytes=0-"}`
- **THEN** the player receives only `Referer: https://a.io`

### Requirement: Loading and failure states
The picker SHALL show a loading indicator until the first group arrives, and keep a smaller indicator while other add-ons are still pending. If every queried add-on fails or returns no streams, it SHALL show an error naming the add-ons tried and a Retry action that repeats the whole query. B SHALL return to the details screen.

#### Scenario: All fail
- **WHEN** both queried add-ons time out
- **THEN** the picker shows "No streams found" with the two add-on names and a Retry button
