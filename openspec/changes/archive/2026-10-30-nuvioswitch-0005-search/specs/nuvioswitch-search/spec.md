## ADDED Requirements

### Requirement: Query input
The Search tab SHALL provide a query field that opens the system software keyboard (swkbd on Switch, borealis IME on desktop) when activated. The search SHALL run when the keyboard input is confirmed; queries are trimmed, and a trimmed query shorter than 2 characters SHALL NOT run a search.

#### Scenario: Search submitted
- **WHEN** the user activates the query field, types "dune" and confirms
- **THEN** the field shows "dune" and a search for "dune" starts

#### Scenario: Too short
- **WHEN** the user confirms the query "d"
- **THEN** no request is sent and the results area stays as it was

#### Scenario: Keyboard cancelled
- **WHEN** the user opens the keyboard and cancels it
- **THEN** the previous query and results are kept

### Requirement: Searchable catalogs
A search SHALL query every catalog of every enabled add-on that supports the `search` extra and declares no other required extra. Each catalog is requested with only `search=<query>` as extra, using the catalog URL rules from `nuvioswitch-home-catalogs`, for the first page only. Requests SHALL run in parallel.

#### Scenario: Catalog selection
- **WHEN** add-on A has catalog `top` (no search extra), catalog `search-movies` (search optional) and catalog `genre-search` (search plus a required `genre`)
- **THEN** only `search-movies` is queried

#### Scenario: Request URL
- **WHEN** the query is "the office" and the catalog is `series`/`imdb-search` on `https://example.com/abc`
- **THEN** the request URL is `https://example.com/abc/catalog/series/imdb-search/search=the%20office.json`

#### Scenario: No searchable catalogs
- **WHEN** no enabled add-on has a searchable catalog
- **THEN** the screen shows a message saying no installed add-on supports search, and sends no request

### Requirement: Result rows
Results SHALL be shown as one horizontal row per searched catalog, titled with the catalog name and add-on name, ordered by add-on order and then by catalog order within the add-on. A row SHALL show a loading state until its response arrives. Rows whose response is empty or fails SHALL be removed. Selecting a poster SHALL open the details screen for that item, passing the item's type, ID and the add-on it came from.

#### Scenario: Rows fill in as responses arrive
- **WHEN** a search queries three catalogs and the second answers first
- **THEN** the second row shows its posters while the first and third still show loading, and row order does not change

#### Scenario: Empty and failed catalogs
- **WHEN** one catalog returns `{"metas": []}` and another fails with HTTP 500
- **THEN** neither row is shown

#### Scenario: Nothing found
- **WHEN** every searched catalog returns no results or fails
- **THEN** the screen shows "No results for \"<query>\""

### Requirement: Stale results discarded
Starting a new search SHALL cancel tracking of the previous one, and responses that belong to an earlier search SHALL NOT change the displayed rows.

#### Scenario: Late response
- **WHEN** the user searches "alien", then searches "aliens" before all "alien" responses arrive
- **THEN** late "alien" responses are ignored and only "aliens" rows are shown

### Requirement: Recent searches
The app SHALL keep a per-profile list of up to 8 recent queries, newest first, de-duplicated case-insensitively (a repeated query moves to the top and keeps its newest spelling). A query SHALL be saved only when its search finished with at least one result. When the query field is empty, the list SHALL be shown; selecting an entry SHALL run that search, X SHALL remove the focused entry, and a "Clear" action SHALL remove all entries.

#### Scenario: Saved after results
- **WHEN** the user searches "Dune" and at least one row has results
- **THEN** "Dune" is at the top of recent searches, and survives an app restart

#### Scenario: Not saved without results
- **WHEN** a search for "zzqx" returns nothing from every catalog
- **THEN** recent searches are unchanged

#### Scenario: De-duplication and limit
- **WHEN** recent searches are 8 entries including "dune", and the user searches "DUNE" with results
- **THEN** the list still has 8 entries, "DUNE" is first, and the older "dune" entry is gone

#### Scenario: Remove one
- **WHEN** focus is on a recent search and the user presses X
- **THEN** that entry is removed from the list and from storage
