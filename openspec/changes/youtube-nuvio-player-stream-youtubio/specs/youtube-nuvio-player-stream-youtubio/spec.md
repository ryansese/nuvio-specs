## ADDED Requirements

### Requirement: Nuvio Player stream
YouTubio SHALL return a stream named exactly `Nuvio Player` for every `youtube.com` video for which it returns `Stremio Player`. The stream SHALL have no `url`, its `externalUrl` SHALL equal the video's watch URL, its `description` SHALL be `Click to watch using Nuvio's built-in Youtube Player`, and its `behaviorHints.filename` SHALL match the other streams of that video.

#### Scenario: YouTube video
- **WHEN** the stream route is called for a `youtube.com` video
- **THEN** the response contains `Nuvio Player` with `externalUrl` equal to the watch URL and the description `Click to watch using Nuvio's built-in Youtube Player`

#### Scenario: Non-YouTube source
- **WHEN** the stream route is called for a video from another site
- **THEN** no `Nuvio Player` stream is returned

### Requirement: Existing streams are unchanged
YouTubio SHALL keep returning every stream it returned before this change with the same names and fields, including `YT-DLP Player <resolution>`, `SB Player <resolution>`, `Stremio Player`, `External Player`, `YT-DLP Channel` and `External Channel`. The manifest, meta links and configure page SHALL be unchanged.

#### Scenario: Stream set otherwise identical
- **WHEN** the stream route is called for a video
- **THEN** the response equals the pre-change response plus exactly one `Nuvio Player` stream
