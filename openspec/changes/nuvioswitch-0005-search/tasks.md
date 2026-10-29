## 1. Search logic

- [ ] 1.1 Add `src/search/search_service.{hpp,cpp}`: `targets(addons)` returning enabled add-ons' searchable catalogs (search supported, no other required extra) in add-on then catalog order
- [ ] 1.2 Add `src/search/search_history.{hpp,cpp}`: trim, minimum length 2, case-insensitive de-duplication, newest first, cap 8, remove one, clear; load and save through 0002 storage as `search_history`
- [ ] 1.3 Unit tests for target selection (search optional, required search, required genre, disabled add-on, ordering) and for history rules
- [ ] 1.4 Unit test that the catalog URL for a search query matches `.../catalog/<type>/<id>/search=<encoded>.json`, keeping any base query string after `.json`

## 2. Search screen

- [ ] 2.1 Add `resources/xml/tabs/search.xml` and `src/tabs/search_tab.{hpp,cpp}`; register the view and replace the Search placeholder in `main.xml`
- [ ] 2.2 Query field opens the IME via `ImeManager::openForText`; on confirm, start a search; on cancel, keep the current state
- [ ] 2.3 Run one catalog request per target in parallel, with a generation counter to ignore stale results; show loading rows in target order
- [ ] 2.4 Fill rows on success, remove empty or failed rows, show "No results" or "No add-on supports search" messages
- [ ] 2.5 Recent searches list when the query is empty: select to search, X to remove, Y to clear all; save history only after a search with results
- [ ] 2.6 Selecting a poster opens the details screen with type, ID and source add-on (no-op until 0006 lands)
- [ ] 2.7 Add the i18n strings to `resources/i18n/en-US/app.json`

## 3. Verification

- [ ] 3.1 `ctest --test-dir build_desktop` passes, including the new search tests
- [ ] 3.2 Desktop: with Cinemeta installed, search "dune" and see movie and series rows; search again quickly and check no stale rows appear; check history persists after restart
- [ ] 3.3 Switch: build the `.nro`, search with swkbd on hardware or in an emulator, and check rows, history removal and clear
