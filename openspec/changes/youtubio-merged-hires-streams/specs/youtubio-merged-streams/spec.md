## ADDED Requirements

### Requirement: Merged stream entries
When merged streams are enabled, the addon SHALL add one stream entry named `Merged <height>p` for each distinct video height that has a video-only format and a compatible audio-only format, in addition to all existing stream entries.

#### Scenario: Higher-resolution formats are offered
- **WHEN** merged streams are enabled and yt-dlp returns video-only formats at 1080p and 720p plus an audio-only format
- **THEN** the stream list contains `Merged 1080p` and `Merged 720p` entries alongside the existing entries

#### Scenario: Existing streams are unchanged
- **WHEN** merged streams are enabled
- **THEN** the existing `YT-DLP Player`, `SB Player`, `Stremio Player`, `External Player` and channel entries are still returned with the same fields as before

#### Scenario: No audio-only format available
- **WHEN** a video has video-only formats but no audio-only format
- **THEN** no `Merged` entries are emitted for that video

### Requirement: Standard stream shape
Each merged stream entry SHALL be a standard stream object with a single `url`, `name`, `description`, and `behaviorHints` containing `bingeGroup` set to the entry name and `notWebReady: true`, so that Stremio and Nuvio can play it with no client-specific handling.

#### Scenario: Consumed by Stremio and Nuvio
- **WHEN** either client requests streams for a video with merged streams enabled
- **THEN** the merged entries contain only standard stream fields and an `http(s)` `url` pointing at the addon's merged playlist route (`hls` mode) or mux route (`mux` mode)

### Requirement: Format pairing
The addon SHALL pair each video-only format with the best available audio-only format, preferring H.264 (`avc1`) video and AAC (`mp4a`) audio, and SHALL fall back to other codecs only when no H.264 format exists at that height.

#### Scenario: H.264 preferred
- **WHEN** both H.264 and VP9 video-only formats exist at 1080p
- **THEN** the `Merged 1080p` entry uses the H.264 format with an AAC audio format

#### Scenario: Codec fallback
- **WHEN** only VP9 or AV1 video-only formats exist at a height
- **THEN** the entry for that height uses the available video format with the best audio-only format

### Requirement: Merged HLS playlist route
In `hls` mode the addon SHALL expose `GET /merged.m3u8` returning an HLS master playlist that references the selected video-only HLS media playlist and audio-only HLS media playlist, and SHALL NOT proxy or re-serve any media segments.

#### Scenario: Master playlist content
- **WHEN** a client requests `/merged.m3u8` with the video-only and audio-only HLS playlist URLs, resolution, bandwidth and codecs
- **THEN** the response is an `#EXTM3U` master with an `EXT-X-MEDIA:TYPE=AUDIO` entry whose `URI` is the audio playlist, and an `EXT-X-STREAM-INF` entry with `AUDIO`, `RESOLUTION` and `BANDWIDTH` whose URI is the video playlist

#### Scenario: No media through the addon
- **WHEN** a player plays a merged entry in `hls` mode
- **THEN** the playlist and segment requests go to googlevideo hosts directly and the addon serves only the master playlist

#### Scenario: Untrusted playlist URL
- **WHEN** a request supplies a URL that is not `https` or not on a googlevideo.com host
- **THEN** the route responds with a 400 status

#### Scenario: Missing HLS pair
- **WHEN** no video-only or no audio-only HLS format exists for a height
- **THEN** no `Merged` entry is emitted for that height in `hls` mode

### Requirement: Delivery mode
The addon SHALL support a per-user delivery mode for merged streams, `hls` (default) or `mux` (a "Proxy Merged Streams" checkbox on the configuration page), and SHALL document that `hls` mode requires the player to share the addon's public IP because googlevideo URLs are IP-bound.

#### Scenario: Default mode
- **WHEN** merged streams are enabled and no mode is set
- **THEN** merged entries point at `/merged.m3u8`

#### Scenario: Proxy mode
- **WHEN** the mode is `mux`
- **THEN** merged entries point at `/mux`

### Requirement: Mux route
In `mux` mode the addon SHALL expose `GET /mux` accepting the video and audio source URLs, and SHALL respond with a single media stream that combines them without re-encoding.

#### Scenario: Successful mux
- **WHEN** a client requests `/mux` with valid video and audio URLs
- **THEN** the response is a fragmented MP4 stream containing both tracks, produced with stream copy

#### Scenario: Client disconnects
- **WHEN** the client closes the connection while a mux is in progress
- **THEN** the addon terminates the ffmpeg process

#### Scenario: Untrusted source URL
- **WHEN** a request supplies a URL that is not `https` or not on an allowed YouTube media host
- **THEN** the addon rejects the request with an error status and does not start ffmpeg

### Requirement: Opt-in configuration
Merged streams SHALL be controlled by a per-user config flag that defaults to off and is exposed in the configuration page.

#### Scenario: Flag off
- **WHEN** the flag is off or unset
- **THEN** no `Merged` entries are returned and behavior is identical to before this change

#### Scenario: Flag on
- **WHEN** the user enables the flag in the configuration page and reinstalls the addon
- **THEN** `Merged` entries appear in stream results

### Requirement: ffmpeg availability
The addon SHALL require `ffmpeg` for merged streams in `mux` mode only, and SHALL emit no `mux` merged entries when `ffmpeg` is unavailable. `hls` mode SHALL NOT depend on `ffmpeg`.

#### Scenario: ffmpeg missing
- **WHEN** the flag is on, the mode is `mux`, and `ffmpeg` is not found on `PATH`
- **THEN** merged entries are omitted, the rest of the stream list is returned normally, and an error is logged
