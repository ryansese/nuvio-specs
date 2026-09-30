## ADDED Requirements

### Requirement: Existing stream lists are unchanged
The streams screen SHALL keep every existing group, card, card name and action exactly as before. A stream whose `externalUrl` is a YouTube URL SHALL still be opened in the system browser from its original card, `ytId`-only streams SHALL remain unlisted, and autoplay SHALL be unaffected.

#### Scenario: External Player card
- **WHEN** the user selects a YouTube `externalUrl` card (for example YouTubio's `External Player`) in its original group
- **THEN** the link opens in the system browser as before

#### Scenario: Group titles
- **WHEN** a group's title is derived from its first stream
- **THEN** the title is not changed by this feature

### Requirement: Internal Player tab
The streams screen SHALL show an additional tab named `Internal Player` when the stream list contains a YouTube `externalUrl` stream (watch, shorts, embed, live or `youtu.be`). The tab SHALL list copies of the direct-URL streams of each group that carries such a stream (for YouTubio, the `YT-DLP Player <resolution>` entries) and one entry per add-on and YouTube video. The tab SHALL be absent when there is nothing to list, and the originals SHALL remain in their own groups.

#### Scenario: YouTubio result
- **WHEN** an add-on returns `YT-DLP Player 640x360` (direct URL) and `External Player` (YouTube `externalUrl`) for a video
- **THEN** the tab bar has an `Internal Player` tab listing both the `YT-DLP Player 640x360` entry and one in-app entry for the video, and the original group still lists all its cards

#### Scenario: Duplicate YouTube links
- **WHEN** the same add-on returns two `externalUrl` streams for the same YouTube video id
- **THEN** the Internal Player tab lists the video once

#### Scenario: No YouTube content
- **WHEN** no group carries a YouTube `externalUrl` stream
- **THEN** no Internal Player tab is shown

### Requirement: In-app resolution and internal playback
When the user selects an in-app YouTube entry in the Internal Player tab, the app SHALL resolve it with the in-app YouTube resolver and play it in the internal libmpv player on macOS, Windows and Linux. Selecting a direct-URL entry in the tab SHALL play it in the internal player as it does in its original group.

#### Scenario: Manual selection
- **WHEN** the user selects an in-app YouTube entry in the Internal Player tab
- **THEN** the video is resolved and the player screen opens and starts playback

#### Scenario: Windows
- **WHEN** an in-app YouTube entry is selected on Windows, where trailer playback mode is external
- **THEN** stream playback still uses the internal player

#### Scenario: Resolution failure
- **WHEN** the resolver returns no result
- **THEN** the app shows an error message and does not open the player

### Requirement: 1080p preference
In-app YouTube playback SHALL prefer the tallest available stream at or below 1080p and SHALL fall back to the next lower available resolution when 1080p is not offered. If only streams above 1080p exist, the lowest available SHALL be used. Trailer playback SHALL not be affected.

#### Scenario: 1080p available
- **WHEN** the video offers 1080p and higher resolutions
- **THEN** playback uses 1080p

#### Scenario: 1080p not available
- **WHEN** the highest resolution at or below 1080p is 720p
- **THEN** playback uses 720p

### Requirement: Separate audio is played
When the resolver returns separate video and audio URLs, the desktop player SHALL load both so the video plays with sound. If the platform bridge cannot load a separate audio source, the app SHALL request a muxed or HLS source instead.

#### Scenario: Separate audio URL
- **WHEN** the resolved source has a video URL and an audio URL
- **THEN** mpv is given the video URL and the audio URL as an external audio file (`audio-files`, without splitting the URL on `:`), and audio is audible

#### Scenario: Muxed source
- **WHEN** the resolved source is a single muxed or HLS URL
- **THEN** it plays without an additional audio file

### Requirement: Player starts on macOS
The macOS player bridge SHALL set the numeric locale to `C` before creating libmpv so that playback starts regardless of the user's locale.

#### Scenario: Non-C process locale
- **WHEN** the process locale is not `C` and playback is started
- **THEN** the player is created and playback starts instead of staying on the loading screen

### Requirement: Throttling resilience
Playback of resolved googlevideo URLs SHALL not stall due to throttled unchunked downloads. Where the desktop player has no chunked-read mechanism, the design SHALL specify the mitigation (for example an HLS source, range-chunked requests, or mpv demuxer or network options).

#### Scenario: Long video playback
- **WHEN** a resolved YouTube video longer than 10 minutes is played
- **THEN** playback proceeds without repeated stalling caused by throttling

### Requirement: Resolved URLs are not persisted
The app SHALL NOT save a resolved YouTube playback URL to the stream link cache as the last-used link.

#### Scenario: Replaying a YouTube stream
- **WHEN** the user selects the same in-app YouTube entry again later
- **THEN** it is resolved again rather than played from a cached resolved URL
