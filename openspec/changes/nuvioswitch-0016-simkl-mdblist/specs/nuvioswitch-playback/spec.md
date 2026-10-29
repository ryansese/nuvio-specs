## ADDED Requirements

### Requirement: Scrobbles delivered to every connected tracker
Each scrobble event SHALL be delivered independently to every tracker connected for the active profile (Trakt, Simkl, MDBList). A failure or slow response from one tracker SHALL NOT delay or cancel delivery to the others. Seek restarts (stop, then start) SHALL go only to trackers that ask for them, which is only Trakt, as in NuvioTV's `TrackingSeekScrobblePolicy`.

#### Scenario: Two trackers, one down
- **WHEN** Trakt and Simkl are connected and Simkl times out on a pause event
- **THEN** Trakt still receives its `scrobble/stop` for the pause

#### Scenario: Seek with Simkl only
- **WHEN** only Simkl is connected and the user seeks from 60% to 20%
- **THEN** no stop or start is sent to Simkl for the seek
