## ADDED Requirements

### Requirement: TMDB enrichment of details
When TMDB is enabled in Settings, the details screen SHALL first show the add-on meta, then merge the TMDB enrichment in when it arrives, per settings group, as in `MetaDetailsViewModel.enrichMeta`:
- **Artwork:** TMDB backdrop and logo replace the add-on's when present.
- **Basic info:** the localised title and description replace the add-on's when present; TMDB genres replace them when not empty; the TMDB rating is shown with its own TMDB label.
- **Details:** runtime, status, age rating, country and language replace the add-on's when present.
- **Credits:** director, writer and a cast row with photos.
- **Episodes:** each episode's title, overview, thumbnail and runtime come from TMDB when present, matched by season and episode number.

Fields TMDB doesn't provide SHALL keep the add-on values.

#### Scenario: Logo added
- **WHEN** TMDB is on and Cinemeta's meta for a movie has no logo but TMDB has an English logo
- **THEN** the hero shows the TMDB logo instead of the text title

#### Scenario: Artwork toggle off
- **WHEN** TMDB is on with Artwork turned off
- **THEN** the backdrop and logo stay as the add-on provides them, while other enabled groups still apply

#### Scenario: Episode stills
- **WHEN** a series' add-on episodes have no thumbnails and episode enrichment is on
- **THEN** the episode list shows TMDB stills for every episode TMDB knows

### Requirement: tmdb: content ids
For content ids starting with `tmdb:`, the details screen SHALL first try to map them to an IMDb id with `{movie|tv}/{id}/external_ids`, waiting at most 5 seconds, and use the IMDb id for the add-on meta lookup. If no add-on returns meta and TMDB is available, it SHALL build the details page from the TMDB enrichment alone, with no episode list or streams of its own, as `tryApplyTmdbFallbackMeta` does.

#### Scenario: tmdb id mapped to IMDb
- **WHEN** a catalog item has id `tmdb:1396` with type series
- **THEN** the details screen resolves `tt0903747` and loads the meta from Cinemeta

#### Scenario: TMDB-only fallback
- **WHEN** no installed add-on returns meta for `tmdb:999999`, but TMDB knows the id
- **THEN** a details page shows TMDB's title, artwork, description and cast
