## ADDED Requirements

### Requirement: YouTube stream detection
The app SHALL treat a stream as a resolvable YouTube stream when it has a `ytId` and no directly playable URL, or when it has no `url` and its `externalUrl` is a YouTube watch, shorts, embed or `youtu.be` URL. Torrent and direct-debrid streams SHALL NOT be treated as YouTube streams.

#### Scenario: ytId stream without url
- **WHEN** an add-on returns a stream with `ytId` set and no `url`
- **THEN** the stream is resolvable as a YouTube stream

#### Scenario: YouTube externalUrl stream
- **WHEN** an add-on returns a stream with no `url` and `externalUrl` equal to `https://www.youtube.com/watch?v=<id>`
- **THEN** the stream is resolvable as a YouTube stream with that video id
- **AND** the stream is not treated as an external (browser) stream

#### Scenario: Non-YouTube externalUrl
- **WHEN** an add-on returns a stream with no `url` and an `externalUrl` that is not a YouTube URL
- **THEN** the stream keeps the existing external (browser) behavior

### Requirement: In-app resolution and internal playback
When a resolvable YouTube stream is selected and the player preference resolves to the internal player, the app SHALL resolve the video to a playable URL on-device and play it in the internal player without opening a browser.

#### Scenario: Successful playback
- **WHEN** the user selects a resolvable YouTube stream and the preference is Internal
- **THEN** a resolving indicator is shown, the video is resolved on-device, and the player screen starts playback of the resolved URL

#### Scenario: Resolution failure
- **WHEN** on-device resolution fails
- **THEN** the app shows the YouTube resolution failure message and does not navigate to the player

### Requirement: Player preference is respected
The app SHALL apply the existing player preference (Internal, External, Ask every time) to YouTube streams. When launching an external player, the app SHALL pass the resolved playable URL, not the YouTube watch page.

#### Scenario: External preference
- **WHEN** the preference is External and the user selects a resolvable YouTube stream
- **THEN** the stream is resolved first and the resolved URL is handed to the external player

#### Scenario: Ask every time
- **WHEN** the preference is Ask every time
- **THEN** the choice dialog is shown and the chosen option uses the resolved URL

### Requirement: Autoplay and in-player switching
Resolvable YouTube streams SHALL be eligible for autoplay selection and SHALL be resolved when switching source or episode from inside the player.

#### Scenario: Autoplay
- **WHEN** autoplay selects a resolvable YouTube stream
- **THEN** it is resolved and played like a manual selection

#### Scenario: Switching source in player
- **WHEN** the user switches to a YouTube stream from the player's source list
- **THEN** the stream is resolved and playback switches to it

### Requirement: Resolved URLs are not persisted
The app SHALL NOT store a resolved YouTube playback URL as the last-used or saved link, because it expires.

#### Scenario: Reopening a previously played YouTube stream
- **WHEN** the user plays a YouTube stream and later selects it again
- **THEN** the video is resolved again rather than reusing the earlier resolved URL

### Requirement: Flavor gating
In-app YouTube resolution SHALL be enabled only where in-app YouTube playback is enabled by the build flavor policy (`full`). Where it is disabled (`playstore`), the app SHALL keep opening YouTube streams via the external URL.

#### Scenario: Playstore flavor
- **WHEN** the `playstore` flavor selects a YouTube stream
- **THEN** the YouTube watch page is opened externally as today
