## ADDED Requirements

### Requirement: YouTube stream parsing
The stream parser SHALL read `ytId` (and `yt_id:`-prefixed ids) from add-on stream responses and SHALL NOT discard a stream solely because it has no `url`, `infoHash` or `externalUrl` when `ytId` is present. A stream whose `externalUrl` is a YouTube watch, shorts, embed or `youtu.be` URL SHALL be classified as a YouTube stream.

#### Scenario: ytId-only stream
- **WHEN** an add-on returns `{"ytId": "<id>"}` with no other source fields
- **THEN** the stream appears in the stream list as a YouTube stream

#### Scenario: YouTube externalUrl
- **WHEN** an add-on returns a stream whose `externalUrl` is `https://www.youtube.com/watch?v=<id>`
- **THEN** the stream is classified as a YouTube stream and is not opened in the browser by default

#### Scenario: Other externalUrl
- **WHEN** an add-on returns a stream with a non-YouTube `externalUrl`
- **THEN** it keeps the existing open-externally behavior

### Requirement: In-app resolution and internal playback
When a YouTube stream is selected, manually or through autoplay, the app SHALL resolve it with the in-app YouTube resolver and start the internal player with the resolved video URL and, when separate, audio URL.

#### Scenario: Manual selection
- **WHEN** the user taps a YouTube stream and the external player setting is off
- **THEN** the video is resolved and the player screen opens with `sourceUrl` and `sourceAudioUrl` set from the result

#### Scenario: Autoplay
- **WHEN** autoplay selects a YouTube stream
- **THEN** it follows the same resolution and internal playback path

#### Scenario: Resolution unavailable or failed
- **WHEN** the resolver returns no result (failure, or a build where the resolver is a stub)
- **THEN** the app shows an error message and, for stub builds, falls back to opening the external URL as today

### Requirement: External player setting is respected
The app SHALL honor `externalPlayerEnabled` and the per-stream "open in internal/external player" action for YouTube streams. External launch SHALL use a directly playable URL that needs no separate audio track; otherwise the app SHALL use the internal player.

#### Scenario: External player enabled with muxed URL
- **WHEN** the external player is enabled and the resolver returns a muxed or HLS URL
- **THEN** the resolved URL is passed to the external player

#### Scenario: External player enabled with separate audio
- **WHEN** the external player is enabled and the resolver returns separate video and audio URLs
- **THEN** the app plays the stream in the internal player instead

### Requirement: Chunked playback on Android
The player launch SHALL carry a flag enabling chunked reads for YouTube media URLs, and the Android player SHALL apply it to avoid throttled downloads. iOS MAY ignore the flag.

#### Scenario: Android playback of a resolved URL
- **WHEN** a resolved googlevideo URL is played on Android
- **THEN** the player uses chunked reads

### Requirement: Resolved URLs are not persisted
The app SHALL NOT save a resolved YouTube playback URL to the stream link cache as the last-used link.

#### Scenario: Replaying a YouTube stream
- **WHEN** the user selects the same YouTube stream again later
- **THEN** it is resolved again rather than played from a cached resolved URL

### Requirement: Build variant gating
The resolver is available only in builds that include the in-app extractor. Builds that stub it (Play Store, App Store) SHALL keep the current behavior for YouTube streams.

#### Scenario: Store build
- **WHEN** a store build selects a YouTube stream
- **THEN** the stream opens via its external URL as today, without an unhandled error
