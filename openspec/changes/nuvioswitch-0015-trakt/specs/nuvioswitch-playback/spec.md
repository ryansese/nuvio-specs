## ADDED Requirements

### Requirement: Tracker scrobble events
The player SHALL emit tracker-neutral scrobble events (start, pause or stop, plus a progress percentage and the item's content type, content id, video id, title, year, season and episode) to every connected tracker. It SHALL follow NuvioTV's rules (`ui/screens/player/PlayerRuntimeControllerPlaybackEvents.kt`):
- **Start** is emitted when playback starts or resumes, unless progress is 80% or more.
- **Pause** is emitted when the user pauses, if a start was sent for the item.
- **Stop** is emitted when the user leaves the player or switches stream or episode, if a start was sent or progress is 80% or more.
- **Completion:** a stop at 80% or more is emitted once per item. Natural end of playback emits stop at 99.5%.
- **Seek:** after a seek that lands between 1% and 80%, trackers that restart on seek get stop and then, if still playing, start.
- **Short streams:** streams of 2:01 or less (placeholder streams, as in 0009) emit nothing.

#### Scenario: Watch to the end
- **WHEN** the user plays an episode from 0% to its natural end
- **THEN** trackers get start at about 0% and one stop at 99.5%

#### Scenario: Resume near the end
- **WHEN** the user resumes a movie saved at 85%
- **THEN** no start is emitted

#### Scenario: Seek back
- **WHEN** the user seeks from 60% back to 20% while playing
- **THEN** trackers that restart on seek get stop at 20% and then start

#### Scenario: Leave mid-way
- **WHEN** the user exits the player at 40% after start was sent
- **THEN** trackers get stop at 40%

### Requirement: Scrobbling never blocks playback
Scrobble delivery SHALL run off the UI thread. It SHALL be fire-and-forget from the player's point of view, and a stop sent on exit SHALL complete even after the player screen closes. Failures SHALL be logged, not shown.

#### Scenario: Offline scrobble
- **WHEN** the network is down and the user pauses
- **THEN** playback isn't affected and no error dialog appears
