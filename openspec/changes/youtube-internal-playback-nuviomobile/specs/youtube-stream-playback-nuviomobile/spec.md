## ADDED Requirements

### Requirement: Existing stream list is unchanged
The streams screen SHALL keep every existing add-on group, card, card name and action as before, except for the `Nuvio Player` card. A stream with no `url` whose `externalUrl` is a YouTube URL and whose name is not exactly `Nuvio Player` (for example `External Player`) SHALL still be opened in the browser, and autoplay, in-player source switching and the player preference SHALL behave as before. No extra tab or group SHALL be added.

#### Scenario: External Player card
- **WHEN** the user selects a YouTube `externalUrl` card named `External Player`
- **THEN** the link opens in the browser as before

#### Scenario: Same URL, different name
- **WHEN** a stream has a YouTube `externalUrl` and any name other than `Nuvio Player`
- **THEN** it keeps the existing external (browser) behavior

### Requirement: Nuvio Player plays in the built-in player
A stream with no `url`, the exact name `Nuvio Player` and an `externalUrl` that is a URL on host `youtu.be`, `youtube.com` or a `*.youtube.com` subdomain (after dropping `www.`) with a watch, shorts, embed, live or `youtu.be` path and a valid video id SHALL be recognised as an in-app YouTube stream. Selecting it SHALL resolve the video to a playable URL on-device and play it in the built-in player, without opening a browser or another app and regardless of the internal/external/ask player preference. The card SHALL stay in its add-on group under its own name and description, and torrent and direct-debrid streams SHALL NOT be treated as in-app YouTube streams.

#### Scenario: Successful playback
- **WHEN** the user selects `Nuvio Player` for a YouTube video
- **THEN** a resolving indicator is shown, the video is resolved on-device, and the built-in player starts playback of the resolved URL

#### Scenario: Resolution failure
- **WHEN** on-device resolution fails
- **THEN** the app shows an error message, does not open the browser and does not start the player

#### Scenario: Non-YouTube URL
- **WHEN** a stream named `Nuvio Player` has an `externalUrl` that is not a YouTube video URL, or has a `url`
- **THEN** it keeps the existing behavior for that stream

### Requirement: 1080p preference
In-app YouTube playback of a `Nuvio Player` stream SHALL prefer the tallest available stream at or below 1080p and SHALL fall back to the next lower available resolution when 1080p is not offered. If only streams above 1080p exist, the lowest available SHALL be used. Trailer playback SHALL NOT be affected.

#### Scenario: 1080p available
- **WHEN** the video offers 1080p and higher resolutions
- **THEN** playback uses 1080p

#### Scenario: 1080p not available
- **WHEN** the highest resolution at or below 1080p is 720p
- **THEN** playback uses 720p

### Requirement: Resolved URLs are not persisted
The app SHALL NOT store a resolved YouTube playback URL as the last-used or saved link, because it expires, and autoplay SHALL NOT select a `Nuvio Player` stream.

#### Scenario: Replaying
- **WHEN** the user plays `Nuvio Player` and later selects it again
- **THEN** the video is resolved again rather than reusing the earlier resolved URL

### Requirement: Store variant gating
In-app playback of `Nuvio Player` SHALL be enabled only in builds where the trailer resolver is real (`full`). In the Play Store and App Store variants, where the resolver returns nothing, a `Nuvio Player` card SHALL behave like any other YouTube `externalUrl` card.

#### Scenario: Store variant
- **WHEN** a store variant lists `Nuvio Player`
- **THEN** selecting it opens the YouTube watch page externally as today
