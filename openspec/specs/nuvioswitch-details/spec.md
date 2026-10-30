## Purpose

TBD: Define NuvioSwitch Details: opening the details screen from a poster, meta add-on selection and fallback, meta request URLs and caching, the hero section, seasons and episodes, and hand-off to the stream picker.

## Requirements

### Requirement: Open details from a poster
Selecting a poster anywhere in the app (Home rows, "see all" grids, Search rows) SHALL open the details screen for that item, passing its ID, type and the base URL of the add-on whose catalog it came from. B on the details screen SHALL return to the previous screen with focus on the same poster.

#### Scenario: Open and return
- **WHEN** the user selects the third poster in a Home row and then presses B on the details screen
- **THEN** the details screen closes and focus is back on that third poster

### Requirement: Meta add-on candidate order
To load metadata the app SHALL build an ordered, de-duplicated list of (add-on, type) candidates from enabled add-ons with a `meta` resource, where the requested type is used as given and the inferred type maps `tv` to `series` (or uses `:movie:`, `:series:`, `:tv:`, `:anime:` markers in the ID when the type is not one of movie, series, channel, anime):
1. add-ons whose `meta` resource supports the requested type and whose ID prefixes match the ID, in add-on order;
2. the same for the inferred type, when it differs;
3. the first `meta` add-on whose ID prefixes match, using whichever of the two types it supports;
4. only if the list is still empty: add-ons supporting the requested type that declare no ID prefixes, then the first `meta` add-on with no ID prefixes;
5. only if the list is still empty: add-ons that list the requested type in their manifest `types` and have a `meta` resource.

ID prefixes are taken from the `meta` resource if it declares any, otherwise from the add-on; no prefixes means any ID matches. Prefix and type matching SHALL be case-insensitive. Candidates SHALL be tried one at a time and the first response with a non-null `meta` SHALL be used.

#### Scenario: Prefix match wins over order
- **WHEN** add-on A (first) supports `meta` for `series` with prefix `kitsu:` and add-on B (second) supports `meta` for `series` with prefix `tt`, and the item is `series`/`tt0903747`
- **THEN** B is requested first and A is not requested

#### Scenario: Fallback to next candidate
- **WHEN** the first candidate returns HTTP 404 and the second returns a valid `meta`
- **THEN** the second candidate's metadata is shown

#### Scenario: tv type
- **WHEN** an item of type `tv` is opened and only an add-on supporting `series` matches
- **THEN** that add-on is requested with type `series`

### Requirement: Source add-on fallback
If no candidate returns metadata, the app SHALL request the item from the add-on whose catalog it came from (when known, enabled and different) using `{base}/meta/{type}/{id}.json`. If that also fails, the screen SHALL show an error with a Retry action.

#### Scenario: Only the source add-on has it
- **WHEN** no meta candidate returns metadata but the source add-on does
- **THEN** the details screen shows the source add-on's metadata

#### Scenario: Not found anywhere
- **WHEN** every attempt fails
- **THEN** the screen shows "Couldn't load details" and a Retry button that runs the lookup again

### Requirement: Meta request URL
Meta requests SHALL use `{basePath}/meta/{type}/{id}.json{baseQuery}`, where type and ID are encoded like Java's `URLEncoder` (`:` becomes `%3A`) with `+` replaced by `%20`, as NuvioTV's `encodePathSegment` does, and any `?query` from the add-on base URL is appended after `.json`.

#### Scenario: Configured add-on
- **WHEN** the add-on base URL is `https://x.io/cfg?token=a` and the item is `movie`/`tt1160419`
- **THEN** the request URL is `https://x.io/cfg/meta/movie/tt1160419.json?token=a`

### Requirement: Meta cache
Successful meta lookups SHALL be cached in memory per (type, ID), keeping at most 32 entries and evicting the least recently used. Each entry SHALL expire after the response's `Cache-Control` `max-age`; with no header it SHALL expire after 6 hours; with `no-cache`, `no-store`, or a max-age below 5 minutes it SHALL expire after 5 minutes. A lookup for a (type, ID) already in flight SHALL wait for that lookup instead of sending new requests.

#### Scenario: Reopened quickly
- **WHEN** the user opens a title, goes back and opens it again within its TTL
- **THEN** no meta request is sent the second time

#### Scenario: no-store floor
- **WHEN** a meta response has `Cache-Control: no-store`
- **THEN** it is reused for 5 minutes and refetched after that

### Requirement: Hero section
The details screen SHALL show the backdrop (falling back to the poster), the logo image or else the title text, release info, runtime, IMDb rating, genres, description, and director and cast when present, plus a focused Play button. Missing fields SHALL be omitted without leaving gaps.

#### Scenario: Movie without logo
- **WHEN** a movie's metadata has no `logo`
- **THEN** the title is shown as text and the rest of the hero is unchanged

### Requirement: Seasons and episodes
For items whose metadata has `videos`, the screen SHALL list seasons as tabs in ascending order with specials (season 0) last, and the selected season's episodes sorted by episode number. Each episode SHALL show its thumbnail, "S<s>E<e>", title, release date and overview when present. The initially selected season SHALL be the season of the video named by `behaviorHints.defaultVideoId` (if it exists and is not marked unavailable), else the first season greater than 0, else the first season. If no video has a season number, all videos SHALL be shown as one season.

#### Scenario: Specials not selected first
- **WHEN** a series has seasons 0, 1 and 2 and no default video
- **THEN** season 1 is selected and season 0 is still available as the last tab, after season 2

#### Scenario: Unnumbered videos
- **WHEN** an item's videos have no season or episode numbers
- **THEN** they are shown as a single list without season tabs

### Requirement: Hand-off to the stream picker
Play and episode selection SHALL open the stream picker with the content type, content ID (the meta ID), video ID and display data (title, poster, backdrop, logo, season, episode, episode title). For a movie the video ID SHALL be `behaviorHints.defaultVideoId` when present, otherwise the meta ID. For a series, selecting an episode SHALL use that episode's video ID, and Play SHALL use the first episode of the selected season.

#### Scenario: Movie play
- **WHEN** the user presses Play on `movie`/`tt1160419` with no default video ID
- **THEN** the stream picker opens with video ID `tt1160419`

#### Scenario: Episode select
- **WHEN** the user selects S2E3 whose video ID is `tt0903747:2:3`
- **THEN** the stream picker opens with video ID `tt0903747:2:3`, season 2, episode 3
