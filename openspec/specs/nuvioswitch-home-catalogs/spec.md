## Purpose

TBD: Define NuvioSwitch Home: catalog request and pagination rules, which catalogs appear and in what order, the Classic layout with hero and rows, lazy row loading, the See-all grid, and Home's empty and error states.

## Requirements

### Requirement: Catalog request URLs
The app SHALL request catalog pages from `{basePath}/catalog/{type}/{catalogId}...json{?query}`, where `basePath` and `?query` come from the add-on's canonical URL:
- with no extras and skip 0: `/{type}/{id}.json`;
- with no extras and skip > 0: `/{type}/{id}/skip={n}.json`;
- with extras: `/{type}/{id}/{k1=v1&k2=v2...}.json`, extras kept in the order given, each key and value URL-encoded with spaces as `%20`, and `skip={n}` appended last when skip > 0 and not already present.

Catalog requests SHALL use the add-on (permissive TLS) HTTP mode.

#### Scenario: First page without extras
- **WHEN** the first page of Cinemeta's `top` movie catalog is requested
- **THEN** the URL is `https://v3-cinemeta.strem.io/catalog/movie/top.json`

#### Scenario: Second page
- **WHEN** the next page is requested with skip 100
- **THEN** the URL is `https://v3-cinemeta.strem.io/catalog/movie/top/skip=100.json`

#### Scenario: Extras with a configured add-on
- **WHEN** a catalog of add-on `https://example.com?token=X` is requested with extra `genre=Science Fiction` and skip 50
- **THEN** the URL is `https://example.com/catalog/movie/{id}/genre=Science%20Fiction&skip=50.json?token=X`

### Requirement: Catalog item parsing
The app SHALL parse the response's `metas` array into catalog items with at least id, type, name, poster, posterShape, background, logo, description, releaseInfo, imdbRating and genres. Items without a non-blank `id` or `name` SHALL be dropped. An item's type SHALL fall back to the catalog's type. Duplicate ids within a page SHALL be removed, keeping the first. Each item SHALL remember which add-on it came from.

#### Scenario: Bad entries skipped
- **WHEN** a page contains 20 metas, two without a name and one repeating an earlier id
- **THEN** 17 items are shown

### Requirement: Catalog pagination
A catalog SHALL be treated as supporting `skip` when it lists `skip` in `extra`, `extraSupported` or `extraRequired`. The page step SHALL be `pageSize` when positive, else the smallest positive gap between numeric `skip` options (when there are at least two), else 100. After a page with `n` raw items, the next skip SHALL be `skip + n`. The catalog SHALL be considered to have more pages only if it supports `skip` and `n > 0`.

#### Scenario: Paging Cinemeta
- **WHEN** Cinemeta's `top` catalog returns 100 items for skip 0
- **THEN** the next request uses skip 100

#### Scenario: End of catalog
- **WHEN** a page returns 0 items
- **THEN** no further pages are requested for that catalog

#### Scenario: No skip support
- **WHEN** a catalog doesn't declare `skip`
- **THEN** only its first page is requested

### Requirement: Home catalog selection and order
Home SHALL show one row per catalog of each enabled add-on, ordered first by add-on priority and then by the catalog's position in the manifest. It SHALL exclude:
- catalogs whose `search` extra is required;
- catalogs whose manifest sets `showInHome` explicitly to false.

A catalog whose manifest omits `showInHome` SHALL be shown. The same add-on id, type and catalog id SHALL appear only once.

#### Scenario: Default add-ons
- **WHEN** only Cinemeta and OpenSubtitles are installed
- **THEN** Home shows Cinemeta's home catalogs (such as Popular Movies and Popular Series) in manifest order, and nothing from OpenSubtitles, which has no catalogs

#### Scenario: Search-only catalog hidden
- **WHEN** an add-on declares a catalog with `extra: [{"name":"search","isRequired":true}]`
- **THEN** that catalog does not appear on Home

#### Scenario: Add-on order change
- **WHEN** the user moves add-on B above add-on A in the Add-ons tab
- **THEN** Home rebuilds with B's rows before A's

### Requirement: Row presentation and loading
Each row SHALL show:
- the title `<Catalog name with first letter uppercased> - <Type label>`, where the label is "Movie" for `movie`, "Series" for `series`, and the capitalised type otherwise;
- the caption `from <add-on display name>`;
- the items as 126×189 poster cards with a title label.

The first 4 rows SHALL load when Home opens. Each remaining row SHALL show a loading placeholder and load when it comes within one row of the visible area. When focus reaches the last 5 items of a row that has more pages, the next page SHALL be appended. A row whose first page fails or is empty SHALL be hidden.

#### Scenario: Lazy rows
- **WHEN** Home opens with 10 catalogs
- **THEN** only 4 catalog requests are sent at first, and the others are sent as the user scrolls down

#### Scenario: Infinite row
- **WHEN** the user scrolls right to the 96th of 100 items in a row that supports skip
- **THEN** the next page loads and its items are appended without moving focus

#### Scenario: Failed row
- **WHEN** a catalog request times out
- **THEN** its row is removed and the other rows are unaffected

### Requirement: Hero carousel
Above the rows, Home SHALL show a hero carousel of up to 7 items taken from the loaded rows in row order. The 7 slots SHALL be spread evenly across rows, with earlier rows getting the remainder. Items with a background image SHALL be preferred, and the same item SHALL NOT appear twice. The hero SHALL show the focused item's background, logo (or name), release info, rating and description, and SHALL advance automatically when not focused. When no loaded item exists, the hero SHALL be hidden.

#### Scenario: Hero filled from two rows
- **WHEN** two rows have loaded
- **THEN** the hero shows 4 items from the first row and 3 from the second, skipping items without a background where others are available

### Requirement: See-all grid
From a row the user SHALL be able to open a See-all screen for that catalog, through a "See all" card at the end of the row or the row's options. It shows the catalog title and a poster grid that loads further pages as focus nears the end, following the pagination rules. B SHALL return to Home with focus restored on the row.

#### Scenario: Browse whole catalog
- **WHEN** the user opens See all on Popular Movies and scrolls down
- **THEN** new pages load until the catalog reports no more

### Requirement: Poster selection
Selecting a poster on Home or in See all SHALL open the item's details. Until the Details screen exists, a stub showing the item's name, type, id and source add-on SHALL be opened, and B returns to where the user was.

#### Scenario: Open stub
- **WHEN** the user presses A on a poster
- **THEN** the details stub for that item opens, and B returns focus to the same poster

### Requirement: Home states
While no row has loaded, Home SHALL show a loading indicator. When no enabled add-on provides a Home catalog, Home SHALL show "No catalog add-ons installed" with a button that opens the Add-ons tab. When every row has failed, Home SHALL show an error with a Retry action.

#### Scenario: No catalog add-ons
- **WHEN** the only installed add-on is OpenSubtitles
- **THEN** Home shows "No catalog add-ons installed" and a button to the Add-ons tab

#### Scenario: Offline
- **WHEN** the network is unavailable and no catalog responses are cached
- **THEN** Home shows the error state, and Retry reloads the rows
