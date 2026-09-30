## ADDED Requirements

### Requirement: Existing stream lists are unchanged
The streams screen SHALL keep every existing group, card, card name and action exactly as before. A stream whose `externalUrl` is a YouTube URL SHALL still be opened in the browser from its original card, `ytId`-only streams SHALL remain unlisted, and autoplay and the external-player setting SHALL behave as before for existing cards.

#### Scenario: External Player card
- **WHEN** the user selects a YouTube `externalUrl` card in its original group
- **THEN** the link opens in the browser as before

#### Scenario: Group titles
- **WHEN** a group's title is derived from its first stream
- **THEN** the title is not changed by this feature

### Requirement: Internal Player tab
The streams screen SHALL show an additional tab named `Internal Player` when the stream list contains a YouTube `externalUrl` stream (watch, shorts, embed, live or `youtu.be`) and the build includes the in-app resolver. The tab SHALL list copies of the direct-URL streams of each group that carries such a stream (for YouTubio, the `YT-DLP Player <resolution>` entries) and one entry per add-on and YouTube video. The tab SHALL be absent when there is nothing to list, and the originals SHALL remain in their own groups.

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
When the user selects an in-app YouTube entry in the Internal Player tab, the app SHALL resolve it with the in-app YouTube resolver and start the internal player with the resolved video URL and, when separate, audio URL. Selecting a direct-URL entry in the tab SHALL play it as it does in its original group. Tab entries SHALL always use the internal player.

#### Scenario: Manual selection
- **WHEN** the user taps an in-app YouTube entry in the Internal Player tab
- **THEN** the video is resolved and the player screen opens with `sourceUrl` and `sourceAudioUrl` set from the result, even if the external-player setting is on

#### Scenario: Resolution failed
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

### Requirement: Chunked playback on Android
The player launch SHALL carry a flag enabling chunked reads for YouTube media URLs, and the Android player SHALL apply it to avoid throttled downloads. iOS MAY ignore the flag.

#### Scenario: Android playback of a resolved URL
- **WHEN** a resolved googlevideo URL is played on Android
- **THEN** the player uses chunked reads

### Requirement: Resolved URLs are not persisted
The app SHALL NOT save a resolved YouTube playback URL to the stream link cache as the last-used link.

#### Scenario: Replaying a YouTube stream
- **WHEN** the user selects the same in-app YouTube entry again later
- **THEN** it is resolved again rather than played from a cached resolved URL

### Requirement: Build variant gating
The resolver is available only in builds that include the in-app extractor. Builds that stub it (Play Store, App Store) SHALL show no Internal Player tab and SHALL keep the current behavior for YouTube streams.

#### Scenario: Store build
- **WHEN** a store build lists a stream with a YouTube `externalUrl`
- **THEN** no Internal Player tab is shown and the card opens its external URL as today
