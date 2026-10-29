## Context

NuvioTV's search lives in `ui/screens/search/SearchViewModel.kt` (about 1,200 lines). Most of it handles things a gamepad UI on Switch doesn't need: live search while typing (350 ms debounce), title suggestions, an embedded Discover view, posters prefetched as focus moves, and hiding unreleased titles. The parts we keep are:

- **`buildSearchTargets` / `isSearchable`:** every catalog that supports `search` and has no other required extra.
- **`loadCatalog`:** a first-page catalog request with `extraArgs = {search: query}`, through the normal catalog repository.
- **Row assembly:** rows are kept in a `linkedMapOf` in target order. Placeholder rows show until data arrives; rows that come back empty are dropped.
- **A search generation counter:** late responses from an earlier search are ignored.
- **`SearchHistoryDataStore`:** max 8 entries, trimmed, de-duplicated case-insensitively, newest first. History is written only when the search has results (`rememberToHistory && any row non-empty`).

0004 provides the catalog fetch and row views, 0003 the add-on list, 0002 HTTP and storage.

## Goals / Non-Goals

**Goals:**
- Search that sends one request per searchable catalog when the user confirms a query, and fills in the results as they arrive.
- Recent searches that persist per profile.
- Keep the deciding logic (which catalogs to query, history updates) pure and unit-tested.

**Non-Goals:**
- Live search while typing and suggestions. swkbd is a modal keyboard, so there's no keystroke stream to react to.
- Discover (type, catalog and genre chips), which may come in a later spec.
- Paging search rows past the first page. NuvioTV supports it, but most search catalogs return a single page.

## Decisions

**1. Search runs on keyboard confirm only.**
- `brls::Application::getImeManager()->openForText(callback, header, ...)` returns the final string, which is the natural point to search.
- Alternative: copy NuvioTV's debounced live search. Rejected: swkbd gives no intermediate text, and firing a request per catalog per keystroke on Switch Wi-Fi is wasteful.

**2. `SearchService` holds pure planning logic; `SearchTab` does the async work.**
- `SearchService::targets(const std::vector<Addon>&)` returns `(addon, catalog)` pairs in order.
- `SearchHistory` has `add`, `remove` and `clear` on a `std::vector<std::string>`.
- Both sit in `src/search/` without borealis includes, so `NuvioSwitchTests` can compile them.
- `SearchTab` keeps a `generation` counter. Each catalog request captures it, and the `brls::sync` callback drops results whose generation doesn't match.

**3. Rows reuse 0004's catalog row view.**
- Rows go in a vertical `ScrollingFrame` (or `RecyclerFrame` if 0004 uses one for Home), one row per target, created in order with a loading state.
- On success, the row gets its items. On empty or error, it's removed. Order never changes, so focus doesn't jump as results arrive.
- Rows are capped at the catalog's first-page size; there's no "see all" here in this change.

**4. History storage.**
- History is a JSON array in the per-profile storage from 0002, under the store name `search_history`, which mirrors NuvioTV's `SearchHistoryDataStore`.
- It is written only from the UI thread after a search completes with at least one result.

**5. Layout.**
- Top: a focusable query field showing the current query or a hint ("Search movies and shows").
- Below it, when the query is empty: the recent-searches list.
- When a search has run: the result rows. "Clear" is a footer action hint (Y) and X removes one entry. Both are registered as borealis actions on the list.

## Risks / Trade-offs

- [Many add-ons with search catalogs mean many parallel requests] → 0002's worker pool limits concurrency, and results stream in either way. Same behaviour as NuvioTV.
- [swkbd behaves differently on desktop (borealis's own IME dialog)] → The behaviour is the same at the callback level, and is tested on both.
- [The catalog row view from 0004 may assume Home-specific data such as "see all"] → If so, add a flag to hide Home-only actions rather than duplicating the view.
