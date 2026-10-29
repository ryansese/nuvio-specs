## ADDED Requirements

### Requirement: Periodic progress saving
During playback the player SHALL save watch progress:
- every 90 seconds, when the position has moved at least 5 seconds since the last save;
- when playback is paused;
- when the user leaves the player;
- when playback ends naturally.

It SHALL NOT save:
- before the first video frame has rendered;
- when the position is below 1 second;
- when the duration is between 1 ms and 120 999 ms (short placeholder or error streams).

#### Scenario: Save on interval
- **WHEN** a film has been playing for 90 seconds since the last save and the position moved by 90 seconds
- **THEN** a progress entry with the current position and duration is written

#### Scenario: Placeholder stream ignored
- **WHEN** the user plays a 30-second "stream not ready" video to the end
- **THEN** no progress or watched record is written

### Requirement: Resume when starting playback
The player SHALL start at the resume position from the watch progress rules, passed to mpv as the `start` option of `loadfile`. When there is no resume position, it SHALL start at 0.

#### Scenario: Start at saved position
- **WHEN** the player opens an episode saved at 12:30
- **THEN** mpv is asked to load the stream with `start=+750`
