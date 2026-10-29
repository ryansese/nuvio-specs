## ADDED Requirements

### Requirement: Skip interval sources
When playback starts, the app SHALL fetch skip intervals (start and end in seconds, type, provider) in the background:
- **IMDb IDs (`tt…`):**
  - from IntroDB `GET {NUVIO_INTRODB_API_URL}/segments?imdb_id=…`;
  - adding `&season=…&episode=…` for episodes, or `&is_movie=true` for movies.
- **`mal:` IDs:** from AniSkip `GET https://api.aniskip.com/v2/skip-times/{malId}/{episode}?types[]=op&types[]=ed&types[]=recap&types[]=mixed-op&types[]=mixed-ed&episodeLength=0`.

Failures and empty responses SHALL produce no intervals and no error message. Results SHALL be cached in memory per content and episode for the session. When `NUVIO_INTRODB_API_URL` is empty, IntroDB SHALL be skipped.

#### Scenario: Episode with IntroDB data
- **WHEN** an episode of `tt0944947` S1E3 starts and IntroDB returns an intro from 0:35 to 1:20
- **THEN** an interval of type `intro` from 35 to 80 seconds is available to the player

#### Scenario: Service down
- **WHEN** IntroDB returns HTTP 500
- **THEN** playback continues normally with no skip button

### Requirement: IntroDB response mapping
For episodes, IntroDB `intro`, `recap` and `outro` SHALL map to types `intro`, `recap` and `outro`. For movies, `outro` SHALL map to `movie-credits` and `post_credits` to `post-credits`; when they overlap, the credits interval SHALL be trimmed to end where the post-credits scene starts. Times SHALL come from `start_sec`/`end_sec`, or from `start_ms`/`end_ms` divided by 1000. Intervals with a non-finite or negative start, or with an end not after the start, SHALL be dropped.

#### Scenario: Credits overlap a post-credits scene
- **WHEN** a movie has `outro` 110:00–118:00 and `post_credits` 115:00–116:00
- **THEN** the credits interval becomes 110:00–115:00 and the post-credits interval is kept separately

### Requirement: Merging providers
Intervals SHALL be grouped into categories:
- **opening:** `intro`, `op`, `mixed-op`;
- **ending:** `outro`, `ed`, `mixed-ed`, `credits`, `ending`;
- **recap:** `recap`.

Each category SHALL be filled from the highest-priority provider that has it, in the order IntroDB, then AniSkip.

#### Scenario: Partial data from two providers
- **WHEN** IntroDB has only an intro and AniSkip has an op and an ed
- **THEN** the merged result is IntroDB's intro and AniSkip's ed

### Requirement: Skip button
While the playback position is inside an interval that is not a post-credits scene, and the skip-intro setting is on, the player SHALL show a focusable skip button labelled by type ("Skip intro", "Skip recap", "Skip credits"). Pressing it SHALL seek to the interval's end. The button SHALL hide itself 10 seconds after it first appears, and SHALL show again while the controls are visible. Dismissing it with B SHALL hide it for that interval unless the controls are shown.

#### Scenario: Skip the intro
- **WHEN** playback reaches 0:40 inside an intro interval ending at 1:20 and the user presses A on "Skip intro"
- **THEN** playback seeks to 1:20 and the button disappears

### Requirement: Auto-skip
For each segment type in the auto-skip setting (`intro`, `recap`, `outro`, `movie-credits`), the player SHALL seek to the interval's end as soon as the position enters it, once per interval per playback, and SHALL briefly show "Skipped intro" (or the matching type). Post-credits scenes SHALL never be skipped.

#### Scenario: Auto-skip recap
- **WHEN** auto-skip includes `recap` and playback enters a recap interval 0:00–0:45
- **THEN** playback jumps to 0:45 and a "Skipped recap" notice is shown
