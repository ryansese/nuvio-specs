## ADDED Requirements

### Requirement: Continue Watching row
When there is at least one item to show, Home SHALL show a "Continue Watching" row above the catalog rows. The row SHALL contain:
- **In progress:**
  - every in-progress movie;
  - for each series, only its most recently watched in-progress episode.
- **Next up:** for each series with a completed episode and no in-progress episode watched after that completion, its next-up episode.

Items SHALL be sorted by last watched, newest first. When the row would be empty it SHALL be hidden.

#### Scenario: Mixed row
- **WHEN** the user is halfway through a film and has finished S1E3 of a series whose S1E4 has aired
- **THEN** the row shows the film with a progress bar and S1E4 of the series labelled "Next up", newest activity first

#### Scenario: One entry per series
- **WHEN** the user has in-progress entries for S1E2 and S1E5 of the same series
- **THEN** only the most recently watched of the two appears in the row

#### Scenario: Nothing to continue
- **WHEN** there are no in-progress items and no next-up episodes
- **THEN** Home shows no Continue Watching row

### Requirement: Continue Watching actions
Selecting a Continue Watching item SHALL open the stream picker for that movie or episode, resuming in-progress items. The item's options menu (X button) SHALL offer:
- "Start from beginning";
- "Go to details";
- "Remove", which deletes the progress entry, or for next-up items hides the series until it has new progress.

#### Scenario: Remove from row
- **WHEN** the user opens the options for a Continue Watching film and picks "Remove"
- **THEN** the film's progress entry is deleted and it disappears from the row
