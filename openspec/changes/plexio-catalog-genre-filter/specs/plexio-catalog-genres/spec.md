## ADDED Requirements

### Requirement: Manifest advertises genres per catalog
For a configured installation, each section catalog in `manifest.json` SHALL include an extra named `genre` with `isRequired: false` and an `options` list holding the titles of the genres that Plex reports for that library section. The options SHALL be listed in the order Plex returns them, without duplicates. The existing `skip`, `search`, and `sort` extras SHALL remain unchanged.

#### Scenario: Section with genres
- **WHEN** a configured installation requests `manifest.json` and the Plex section `1` reports genres `Action`, `Comedy`, `Science Fiction`
- **THEN** the catalog with id `1` has an extra `{"name": "genre", "isRequired": false, "options": ["Action", "Comedy", "Science Fiction"]}`

#### Scenario: Sections have independent genre lists
- **WHEN** the configuration includes a movie section and a show section with different genres
- **THEN** each catalog's `genre` options list only the genres of its own section

#### Scenario: Unconfigured manifest
- **WHEN** `manifest.json` is requested without a configuration
- **THEN** the manifest has no catalogs, makes no Plex request, and still reports `configurationRequired: true`

### Requirement: Manifest survives genre lookup failures
If Plex can't return the genre list for a section (connection error, timeout, error status, or unauthorized), the manifest SHALL still be returned with HTTP 200 and that section's catalog SHALL be present without a `genre` extra. A section that has no genres SHALL also omit the `genre` extra.

#### Scenario: Plex unreachable while building the manifest
- **WHEN** a configured installation requests `manifest.json` and the Plex server times out on the genre request for section `1`
- **THEN** the response is HTTP 200, catalog `1` is listed with its `skip`, `search`, and `sort` extras, and it has no `genre` extra

#### Scenario: One section fails, another succeeds
- **WHEN** the genre request fails for section `1` and succeeds for section `2`
- **THEN** catalog `1` has no `genre` extra and catalog `2` has its `genre` options

#### Scenario: Section with no genres
- **WHEN** Plex reports an empty genre list for a section
- **THEN** that catalog has no `genre` extra

### Requirement: Genre lists are cached
Genre lists SHALL be cached per Plex server and section, so repeated manifest requests within the cache lifetime don't call Plex again. The cache lifetime SHALL be bounded (no longer than 24 hours) so new genres in the library eventually appear. Failed lookups SHALL NOT be cached.

#### Scenario: Repeated manifest requests
- **WHEN** the same installation requests `manifest.json` twice within the cache lifetime
- **THEN** Plex receives the genre request for each section only once

#### Scenario: Failure is retried
- **WHEN** a genre lookup fails and the manifest is requested again
- **THEN** Plexio requests the genre list from Plex again

### Requirement: Catalog filters by genre
The catalog endpoint SHALL accept a `genre` extra whose value is a genre title. When present, the response SHALL contain only items from that section tagged with that genre. The `genre` extra SHALL combine with `search`, `sort`, and `skip`: filtering applies first, then sorting, then pagination. Without a `genre` extra, catalog behavior SHALL be unchanged.

#### Scenario: Filter by genre
- **WHEN** a client requests `/catalog/movie/1/genre=Comedy.json`
- **THEN** every returned item has `Comedy` in its `genres`, and items without that genre are absent

#### Scenario: Genre combined with sort and paging
- **WHEN** a client requests `/catalog/movie/1/genre=Comedy&sort=Year (desc)&skip=100.json`
- **THEN** the response is the second page (items 101–200) of the section's Comedy items, sorted by year descending

#### Scenario: Genre combined with search
- **WHEN** a client requests `/catalog/movie/1/genre=Comedy&search=night.json`
- **THEN** only Comedy items whose title matches `night` are returned

#### Scenario: Unknown genre
- **WHEN** a client requests a genre title that the section doesn't have
- **THEN** the response is HTTP 200 with an empty `metas` list

#### Scenario: Genre matching ignores case
- **WHEN** a client requests `genre=comedy` and the section has the genre `Comedy`
- **THEN** the response contains the section's Comedy items

### Requirement: Catalog extras are URL-decoded
Each catalog extra value SHALL be URL-decoded before it is used, so values containing spaces, `&`, `=`, or non-ASCII characters are matched as the client sent them.

#### Scenario: Multi-word genre
- **WHEN** a client requests `/catalog/movie/1/genre=Science%20Fiction.json`
- **THEN** the catalog is filtered by the genre `Science Fiction`

#### Scenario: Encoded search value
- **WHEN** a client requests `/catalog/movie/1/search=r%C3%A9sum%C3%A9.json`
- **THEN** the Plex title filter receives `résumé`
