## ADDED Requirements

### Requirement: Load remote images into views
App code SHALL be able to ask for a remote image URL to be shown in a borealis `Image` view. The image SHALL be downloaded off the UI thread and set on the view on the UI thread. While it loads, or if it fails, the view SHALL show a placeholder instead of being blank.

#### Scenario: Poster loads
- **WHEN** a poster cell asks to load `https://images.metahub.space/poster/medium/tt0111161/img`
- **THEN** the cell shows the placeholder, and then the poster once it has downloaded

#### Scenario: Broken image
- **WHEN** the URL returns 404 or data that isn't an image
- **THEN** the placeholder stays, and nothing crashes

### Requirement: Stale loads ignored
When a view gets a new image URL, or is reused for another item, before an earlier load has finished, the earlier result SHALL be discarded and never shown in that view.

#### Scenario: Fast scrolling
- **WHEN** a recycled cell is asked to load image A and then image B before A has finished
- **THEN** only image B is ever shown in that cell

### Requirement: Image caching
Downloaded images SHALL be kept in an in-memory LRU cache, capped at 32 MB of encoded bytes, and in a disk cache under `cache/images/`, capped at 100 MB, with least-recently-used files evicted. A cached image SHALL be shown without a network request.

#### Scenario: Memory hit
- **WHEN** a poster that was shown a moment ago scrolls back into view
- **THEN** it appears without a network request

#### Scenario: Disk hit after restart
- **WHEN** the app restarts and shows a poster it downloaded in the previous session
- **THEN** the poster is read from the disk cache without a network request

### Requirement: Limited concurrent image downloads
No more than 4 image downloads SHALL run at once. Pending loads whose view no longer wants them SHALL be dropped before they start downloading.

#### Scenario: Many posters at once
- **WHEN** a screen asks for 40 posters at once
- **THEN** at most 4 download in parallel, and loads that were cancelled while queued never start
