## ADDED Requirements

### Requirement: Play a stream with libmpv
Selecting a playable stream in the stream picker SHALL open a full-screen player that plays the `PlaybackRequest` URL with libmpv on both Switch and desktop, rendered inside the borealis window, with a loading indicator until the first video frame (or, for audio-only media, playback start).

#### Scenario: HTTP stream plays
- **WHEN** the user selects a stream whose URL is a reachable HTTPS MP4 or MKV
- **THEN** the player opens, shows a spinner, and then shows video with audio

### Requirement: Request headers
When the request has headers, the player SHALL set `http-header-fields` before loading the file to the headers with non-blank key and value, sorted by lowercase key then value, each formatted `Key: Value` with `\` replaced by `\\` and `,` replaced by `\,`, joined by `,`. With no headers it SHALL set `http-header-fields` to an empty string. The same headers SHALL be used for external subtitles that don't carry their own.

#### Scenario: Header escaping
- **WHEN** the headers are `{"User-Agent": "A, B", "Cookie": "x=1"}`
- **THEN** `http-header-fields` is `Cookie: x=1,User-Agent: A\, B`

#### Scenario: Headers cleared between streams
- **WHEN** a stream with headers is played, then the user picks a stream without headers
- **THEN** the second load uses an empty `http-header-fields`

### Requirement: Player options
Before the first load the player SHALL set: `keep-open=yes`, `demuxer-max-bytes` and `demuxer-max-back-bytes` to 64 MiB, `sub-ass-override=no`, `sub-codepage=auto:utf-8`, `sub-use-margins=yes`, `sub-ass-force-margins=yes`, `save-position-on-quit=no`, `volume-max=400` (as NuvioTV's `MPV_MAX_VOLUME_PERCENT`), and `user-agent` to `NuvioSwitch/<version>`. TLS verification for media SHALL follow the networking defaults of `nuvioswitch-networking`. Platform-specific values (`hwdec`, `ao`, `vo`) SHALL be chosen per platform in the design.

#### Scenario: Options applied
- **WHEN** the player is created
- **THEN** reading the properties back from mpv shows `keep-open` = `yes` and `demuxer-max-bytes` = 67108864

### Requirement: Start position
When the request has a start position greater than 0, the player SHALL load the file with `loadfile <url> replace -1 start=+<seconds with 3 decimals>`, passing the per-file options as the fourth argument. Without a start position it SHALL use `loadfile <url> replace`.

#### Scenario: Start at 10 minutes
- **WHEN** a request has start position 600000 ms
- **THEN** the loadfile command arguments are `loadfile`, `<url>`, `replace`, `-1`, `start=+600.000`

### Requirement: External subtitles
When the player opens, it SHALL collect subtitles from the request's own `subtitles` and from every enabled add-on that has a `subtitles` (or `subtitle`) resource matching the type (`tv` treated as `series`) and ID prefixes, querying them in parallel with a 20-second timeout each. The add-on URL SHALL be `{basePath}/subtitles/{type}/{id}.json{baseQuery}`, or `{basePath}/subtitles/{type}/{id}/{params}.json{baseQuery}` when any of `videoHash=<hash>`, `videoSize=<bytes>`, `filename=<encoded>` (in that order, joined by `&`) are known, where `{id}` is the video ID for series and otherwise the video ID or content ID. Entries without a URL SHALL be skipped and missing languages SHALL be `und`. A subtitle SHALL be added to mpv with `sub-add <url> cached <title> <lang>` only when the user selects it.

#### Scenario: Subtitle URL with hash
- **WHEN** the request is `series`/`tt0903747:2:3` with videoHash `abc` and videoSize `123`, and OpenSubtitles v3 is installed at `https://opensubtitles-v3.strem.io`
- **THEN** the request URL is `https://opensubtitles-v3.strem.io/subtitles/series/tt0903747%3A2%3A3/videoHash=abc&videoSize=123.json`

#### Scenario: Slow add-on
- **WHEN** one subtitle add-on hasn't answered after 20 seconds
- **THEN** its request is abandoned and the subtitles from the others are listed

### Requirement: Audio and subtitle selection
The Audio panel SHALL list mpv's audio tracks from `track-list` (title, language, codec, channel count) with the active one marked, and selecting one SHALL set `aid`. The Subtitles panel SHALL list "Off", the embedded subtitle tracks, then the external add-on subtitles grouped by language with the add-on name; selecting an embedded track SHALL set `sid`, selecting an external one SHALL `sub-add` it (once) and select it, and "Off" SHALL set `sid=no`. On each new file the player SHALL reset `aid` and `sid` to `auto`.

#### Scenario: Pick an external subtitle
- **WHEN** the user opens Subtitles and selects an English OpenSubtitles entry
- **THEN** mpv adds that subtitle and shows it, and reopening the panel shows it as active

#### Scenario: Turn subtitles off
- **WHEN** the user selects "Off"
- **THEN** no subtitle is rendered

### Requirement: Gamepad controls
With controls hidden: A SHALL toggle play/pause; Left/Right SHALL seek backward/forward by 10 s for a press, growing to 20 s from the 3rd repeat, 30 s from the 8th and 60 s from the 15th while held; Up, Down or X SHALL show the controls. ZL/ZR SHALL seek −10/+10 s at any time. With controls visible: the D-pad SHALL move focus between the seek bar and the buttons (Play/Pause, Subtitles, Audio, Sources), Left/Right on the focused seek bar SHALL seek, and the controls SHALL hide after 4 seconds without input while playing. B SHALL hide visible controls or, when they are hidden, close the player and return to the stream picker.

#### Scenario: Held seek accelerates
- **WHEN** the user holds Right with controls hidden, producing seek events with repeat counts 0, 1, 2, … 16
- **THEN** counts 0–2 seek 10 s each, counts 3–7 seek 20 s, counts 8–14 seek 30 s and counts 15 and above seek 60 s

#### Scenario: Exit
- **WHEN** controls are hidden and the user presses B
- **THEN** playback stops, mpv is shut down, and the stream picker is shown

### Requirement: Playback information
The controls SHALL show the content title (and "S<s>E<e> · <episode title>" for episodes), the elapsed time, the remaining time, a seek bar with the buffered range, and a paused indicator. Position and duration SHALL come from mpv properties observed through mpv events.

#### Scenario: Time display
- **WHEN** a 1:30:00 film is at 0:45:10
- **THEN** the controls show "45:10" elapsed and "-44:50" remaining

### Requirement: Sources action
The Sources button SHALL stop playback and return to the stream picker with the same results, so the user can pick another stream.

#### Scenario: Change source
- **WHEN** the user presses Sources and selects a different stream
- **THEN** the new stream starts playing in a new player

### Requirement: Errors and teardown
When mpv reports that the file failed to load or playback ended with an error, the player SHALL show an error overlay with the reason and two actions: "Try another stream" (back to the picker) and "Back". Leaving the player by any route SHALL stop playback, free the render context and destroy the mpv handle before the activity is removed, without crashing or leaking between plays.

#### Scenario: Dead link
- **WHEN** the stream URL returns HTTP 404
- **THEN** the error overlay appears instead of a frozen spinner

#### Scenario: Repeated plays
- **WHEN** the user starts and exits playback ten times in a row on Switch
- **THEN** the app keeps running and memory use returns to its level before the first play
