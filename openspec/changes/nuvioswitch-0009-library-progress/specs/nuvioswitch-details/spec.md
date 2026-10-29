## ADDED Requirements

### Requirement: Library toggle on Details
The Details screen SHALL show an "Add to library" button that changes to "Remove from library" when the item is saved, and updates immediately when pressed.

#### Scenario: Toggle
- **WHEN** the user presses "Add to library" on a series
- **THEN** the series is saved and the button now reads "Remove from library"

### Requirement: Watched state on Details
For a movie, the Details screen SHALL show "Mark as watched" or "Mark as unwatched". In the episode list, each watched episode SHALL show a check mark, and each in-progress episode a progress bar. An episode's options (X button) SHALL offer "Mark as watched" or "Mark as unwatched".

#### Scenario: Episode marks
- **WHEN** S1E1 is watched and S1E2 is at 40%
- **THEN** the episode list shows a check on S1E1 and a 40% progress bar on S1E2

### Requirement: Resume label on the play button
When the item (for series, the next episode to play) has an in-progress entry, the main play button SHALL read "Resume" followed by the time left, for example "Resume · 52m left". For series without one, it SHALL read "Play S{n}E{m}", naming the next-up episode, or the first episode when nothing has been watched.

#### Scenario: Series play button
- **WHEN** the user has finished S1E3 of a series and has no in-progress episode
- **THEN** the play button reads "Play S1E4"
