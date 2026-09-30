## ADDED Requirements

### Requirement: Existing stream list is unchanged
The streams screen SHALL keep every existing add-on group, card, card name and action exactly as before. A stream with no `url` whose `externalUrl` is a YouTube URL SHALL still be opened in the browser from its original card, `ytId` streams SHALL keep their current resolve-and-play behavior, and autoplay, in-player source switching and the player preference SHALL behave as before for existing cards.

#### Scenario: External Player card
- **WHEN** the user selects a YouTube `externalUrl` card in its original group
- **THEN** the link opens in the browser as before

#### Scenario: ytId stream
- **WHEN** the user selects a stream that carries a `ytId`
- **THEN** it is resolved and played exactly as before this change

### Requirement: Internal Player tab
The streams screen SHALL show an additional `Internal Player` entry in the add-on filter chips when the stream list contains a stream with no `url` whose `externalUrl` is a YouTube watch, shorts, embed, live or `youtu.be` URL and in-app YouTube resolution is enabled. The tab SHALL list copies of the direct-URL streams of each add-on that returned such a stream (for YouTubio, the `YT-DLP Player <resolution>` entries) and one entry per add-on and YouTube video. The tab SHALL be absent when there is nothing to list, and the originals SHALL remain in their own groups. Torrent and direct-debrid streams SHALL NOT be treated as YouTube streams.

#### Scenario: YouTubio result
- **WHEN** an add-on returns `YT-DLP Player 640x360` (direct URL) and `External Player` (YouTube `externalUrl`) for a video
- **THEN** the filter chips include `Internal Player`, listing both the `YT-DLP Player 640x360` entry and one in-app entry for the video, and the original add-on group still lists all its cards

#### Scenario: Duplicate YouTube links
- **WHEN** the same add-on returns two `externalUrl` streams for the same YouTube video id
- **THEN** the Internal Player tab lists the video once

#### Scenario: Non-YouTube externalUrl
- **WHEN** an add-on returns a stream with an `externalUrl` that is not a YouTube URL
- **THEN** it keeps the existing external (browser) behavior and does not create an Internal Player tab

### Requirement: In-app resolution and internal playback
When the user selects an in-app YouTube entry in the Internal Player tab, the app SHALL resolve the video to a playable URL on-device and play it in the internal player without opening a browser, regardless of the player preference. Selecting a direct-URL entry in the tab SHALL play it as it does in its original group.

#### Scenario: Successful playback
- **WHEN** the user selects an in-app YouTube entry in the Internal Player tab
- **THEN** a resolving indicator is shown, the video is resolved on-device, and the player screen starts playback of the resolved URL

#### Scenario: Resolution failure
- **WHEN** on-device resolution fails
- **THEN** the app shows the YouTube resolution failure message and does not navigate to the player

### Requirement: 1080p preference
In-app YouTube playback from the Internal Player tab SHALL prefer the tallest available stream at or below 1080p and SHALL fall back to the next lower available resolution when 1080p is not offered. If only streams above 1080p exist, the lowest available SHALL be used. Trailer playback and `ytId` playback SHALL not be affected.

#### Scenario: 1080p available
- **WHEN** the video offers 1080p and higher resolutions
- **THEN** playback uses 1080p

#### Scenario: 1080p not available
- **WHEN** the highest resolution at or below 1080p is 720p
- **THEN** playback uses 720p

### Requirement: Resolved URLs are not persisted
The app SHALL NOT store a resolved YouTube playback URL as the last-used or saved link, because it expires.

#### Scenario: Reopening a previously played YouTube stream
- **WHEN** the user plays an in-app YouTube entry and later selects it again
- **THEN** the video is resolved again rather than reusing the earlier resolved URL

### Requirement: Flavor gating
The Internal Player tab SHALL be shown only where in-app YouTube playback is enabled by the build flavor policy (`full`). Where it is disabled (`playstore`), no tab SHALL be shown and the app SHALL keep opening YouTube streams via the external URL.

#### Scenario: Playstore flavor
- **WHEN** the `playstore` flavor lists a stream with a YouTube `externalUrl`
- **THEN** no Internal Player tab is shown and the card opens the YouTube watch page externally as today
