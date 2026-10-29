## ADDED Requirements

### Requirement: Watch progress entries and keys
The app SHALL store one watch progress entry per movie or episode for the active profile. An entry SHALL hold:
- `contentId`, `contentType`, `name`, `poster`, `backdrop`, `logo`;
- `videoId`, `season`, `episode`, `episodeTitle`;
- `position` and `duration` in milliseconds;
- `lastWatched` as epoch milliseconds;
- `addonBaseUrl`.

Its key SHALL be `{contentId}_s{season}e{episode}` when both season and episode are set, and `{contentId}` otherwise.

#### Scenario: Episode key
- **WHEN** progress is saved for `tt0944947` season 1 episode 3
- **THEN** it is stored under the key `tt0944947_s1e3`

#### Scenario: Movie key
- **WHEN** progress is saved for movie `tt0111161` with no season or episode
- **THEN** it is stored under the key `tt0111161`

#### Scenario: Artwork kept on update
- **WHEN** an entry with a poster exists and a new save for the same key has no poster or logo
- **THEN** the stored entry keeps the existing poster, backdrop, logo, name and episode title

#### Scenario: Series-level key removed when an episode is saved
- **WHEN** an episode entry is saved for a content ID that also has a series-level entry keyed by the bare `contentId`
- **THEN** the series-level entry is removed

### Requirement: Recent and archive buckets
Progress entries SHALL be split into a recent bucket of the 200 entries with the newest `lastWatched` (ties broken by key, descending) and an archive bucket with the rest. Both buckets SHALL be read together as one map, with recent entries taking precedence.

#### Scenario: Overflow moves to archive
- **WHEN** a 201st entry is saved and it is the newest
- **THEN** the recent bucket holds the 200 newest entries and the oldest entry is moved to the archive bucket

### Requirement: Progress thresholds
Progress percentage SHALL be `position / duration`, clamped to 0–1, and 0 when duration is unknown. An entry SHALL count as in progress at 2% or more and below 90%, and as completed at 90% or more.

#### Scenario: Completed at 90 percent
- **WHEN** a save has position 5400000 and duration 6000000
- **THEN** the entry counts as completed

#### Scenario: Not started below 2 percent
- **WHEN** a save has position 60000 and duration 6000000 (1%)
- **THEN** the entry is stored but does not count as in progress for Continue Watching

### Requirement: Completed items become watched records
When a save reaches the completed threshold, the app SHALL record a watched item (`contentId`, `contentType`, `title`, `season`, `episode`, `watchedAt`, `poster`). It SHALL remove the progress entry for that key. Within the same playback session, later saves SHALL NOT write an in-progress entry for that key again.

#### Scenario: Finishing an episode
- **WHEN** playback of S1E3 reaches 92% and progress is saved
- **THEN** S1E3 is recorded as watched, its progress entry is removed, and a stale save of 5% from the same session is ignored

### Requirement: Resume position
When playback starts for an item with a saved in-progress entry, the start position SHALL be the saved position clamped to the stream's duration. Items that are completed, or below 2%, SHALL start from the beginning. The user SHALL be able to override this with "Start from beginning".

#### Scenario: Resume a film
- **WHEN** the user plays a film saved at 45 minutes of 120
- **THEN** playback starts at 45:00

#### Scenario: Start over
- **WHEN** the user chooses "Start from beginning" on the same film
- **THEN** playback starts at 0:00 and the old position is overwritten by the next save

### Requirement: Next-up episode resolution
For a series whose furthest completed episode is S/E, the next-up episode SHALL be the first episode after S/E in `Meta.videos`, ordered by season then episode, ignoring season 0. When a season changes, the next episode SHALL be skipped if its release date is missing or in the future. Within a season, an episode with a future or missing release date SHALL be skipped.

#### Scenario: Next episode in the same season
- **WHEN** S1E3 is the furthest watched episode and S1E4 has aired
- **THEN** S1E4 is the next-up episode

#### Scenario: Season rollover not yet aired
- **WHEN** S1E10 is the last episode of season 1, it was watched, and S2E1 releases next month
- **THEN** there is no next-up episode for that series

### Requirement: Watched state toggling
The user SHALL be able to mark a movie or episode as watched or unwatched. Marking as watched SHALL record a watched item and remove its progress entry. Marking as unwatched SHALL remove the watched item.

#### Scenario: Unmark an episode
- **WHEN** the user marks a watched S1E2 as unwatched
- **THEN** S1E2 no longer shows a watched mark and can become a next-up candidate again
