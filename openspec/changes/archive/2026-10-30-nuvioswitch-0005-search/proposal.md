# NuvioSwitch-0005 Search

## Why

After 0004, users can browse Home catalogs but cannot look up a specific title. NuvioTV's Search tab sends the user's query to every add-on catalog that supports searching and shows the results as rows. NuvioSwitch needs the same, using the Switch software keyboard for text input.

## What Changes

- Replace the Search tab placeholder from 0001 with a real Search screen: a query field, a recent-searches list and one result row per searchable catalog.
- Text input opens the system keyboard (swkbd) on Switch and borealis's IME on desktop. There is no live search while typing; the search runs when the keyboard is confirmed.
- **Searchable catalogs:** a catalog qualifies if it supports the `search` extra and requires no other extra (`ui/screens/search/SearchViewModel.kt`, `isSearchable`). Each qualifying catalog of each enabled add-on is queried in parallel with `search=<query>`.
- **Result rows:** rows follow add-on order and then catalog order. Each row shows a loading state until its response arrives, and rows that come back empty or fail are dropped. If every catalog fails or returns nothing, an empty or error message is shown.
- **Recent searches:**
  - up to 8 entries, newest first, de-duplicated case-insensitively (`data/local/SearchHistoryDataStore.kt`);
  - saved per profile, only when a search returned at least one result;
  - can be removed one at a time or all cleared.
- Queries shorter than 2 characters are ignored (`MIN_SEARCH_QUERY_LENGTH` in `ui/screens/search/SearchUiState.kt`).
- When a new search starts, results from the previous one are thrown away as they arrive.
- Choosing a result opens the details screen (0006). Until 0006 lands, it does nothing.

## Capabilities

### New Capabilities
- `nuvioswitch-search`: the Search tab: query input, which catalogs are searched and how, result rows, and recent-search history.

### Modified Capabilities
None.

## Impact

- **Code (NuvioSwitch):**
  - new `src/search/` with the search service (pure logic: picks the catalogs to search and builds the query) and the history store;
  - new `src/tabs/search_tab.{hpp,cpp}` and `resources/xml/tabs/search.xml`;
  - new i18n strings in `resources/i18n/en-US/app.json`.
- **Reuses from earlier changes:**
  - the catalog URL builder and catalog fetch, and the catalog row and poster card views (0004);
  - the add-on list and manifest model, including catalog `extra` parsing (0003);
  - the HTTP client and per-profile JSON storage (0002).
- **Depends on:** 0002, 0003, 0004. Uses 0006 for result navigation once it exists.
- **Out of scope:**
  - Discover (filter chips and genre browse);
  - search suggestions, voice search and live-while-typing search;
  - hiding unreleased content.
