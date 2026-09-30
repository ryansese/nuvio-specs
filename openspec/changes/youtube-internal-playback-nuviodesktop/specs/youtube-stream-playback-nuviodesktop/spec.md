## ADDED Requirements

### Requirement: YouTube stream parsing
The stream parser SHALL read `ytId` (and `yt_id:`-prefixed ids) from add-on stream responses and SHALL NOT discard a stream solely because it has no `url`, `infoHash` or `externalUrl` when `ytId` is present. A stream whose `externalUrl` is a YouTube watch, shorts, embed or `youtu.be` URL SHALL be classified as a YouTube stream.

#### Scenario: ytId-only stream
- **WHEN** an add-on returns `{"ytId": "<id>"}` with no other source fields
- **THEN** the stream appears in the stream list as a YouTube stream

#### Scenario: YouTube externalUrl
- **WHEN** an add-on returns a stream whose `externalUrl` is `https://www.youtube.com/watch?v=<id>`
- **THEN** the stream is classified as a YouTube stream and is not opened in the system browser

### Requirement: In-app resolution and internal playback
When a YouTube stream is selected, manually or through autoplay, the app SHALL resolve it with the in-app YouTube resolver and play it in the internal libmpv player on macOS, Windows and Linux. YouTube streams SHALL always use the internal player because the external player is disabled on desktop.

#### Scenario: Manual selection
- **WHEN** the user selects a YouTube stream
- **THEN** the video is resolved and the player screen opens and starts playback

#### Scenario: Windows
- **WHEN** a YouTube stream is selected on Windows, where trailer playback mode is external
- **THEN** stream playback still uses the internal player

#### Scenario: Resolution failure
- **WHEN** the resolver returns no result
- **THEN** the app shows an error message and does not open the player

### Requirement: Separate audio is played
When the resolver returns separate video and audio URLs, the desktop player SHALL load both so the video plays with sound. If the platform bridge cannot load a separate audio source, the app SHALL request a muxed or HLS source instead.

#### Scenario: Separate audio URL
- **WHEN** the resolved source has a video URL and an audio URL
- **THEN** mpv is given the video URL and the audio URL as an external audio file, and audio is audible

#### Scenario: Muxed source
- **WHEN** the resolved source is a single muxed or HLS URL
- **THEN** it plays without an additional audio file

### Requirement: Throttling resilience
Playback of resolved googlevideo URLs SHALL not stall due to throttled unchunked downloads. Where the desktop player has no chunked-read mechanism, the design SHALL specify the mitigation (for example mpv demuxer or network options, or an HLS source).

#### Scenario: Long video playback
- **WHEN** a resolved YouTube video longer than 10 minutes is played
- **THEN** playback proceeds without repeated stalling caused by throttling

### Requirement: Resolved URLs are not persisted
The app SHALL NOT save a resolved YouTube playback URL to the stream link cache as the last-used link.

#### Scenario: Replaying a YouTube stream
- **WHEN** the user selects the same YouTube stream again later
- **THEN** it is resolved again rather than played from a cached resolved URL
