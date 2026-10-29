## Context

After 0003, `AddonRepository` gives the enabled add-ons in priority order with parsed `CatalogDescriptor`s, and notifies observers when they change. 0002 provides `HttpClient`, the JSON helpers, the HTTP cache and `ImageLoader`.

NuvioTV's Home is spread over a few files:
- `HomeViewModelCatalogPipeline.kt`:
  - selects catalogs: enabled add-ons × `catalogs`, filtered by `shouldShowOnHome()` from `HomeViewModelCatalogUtils.kt`;
  - loads the first `eagerCatalogLoadCount = 4` rows straight away and the rest when they're needed, showing placeholders meanwhile;
  - builds row titles "Name - Type";
  - picks hero items by spreading 7 slots across rows (`slotShuffled`) and preferring items with `hasHeroArtwork()`.
- `CatalogRepositoryImpl.kt`: builds the URL and works out pagination (`hasMore`, `nextSkip`).
- `CatalogMapper.kt`: maps catalog items (`toDomainOrNull`).

The Classic layout's UI is `ClassicHomeContent.kt`, with `HeroCarousel.kt`, `CatalogRowSection.kt` and `ContentCard.kt`.

## Goals / Non-Goals

**Goals:**
- Show the same catalogs and items as NuvioTV for the same add-ons, in the same order.
- Smooth gamepad scrolling on the Switch with many rows and posters, within its memory limits.
- A catalog data layer and poster cell that Search (0005) can reuse without changes.

**Non-Goals:**
- Modern and Grid layouts, the layout-choice screen, and Essential/Advanced modes.
- Choosing hero catalogs, reordering or disabling catalogs, custom catalog titles, Collections, and the "hide unreleased" filter. These may come with Settings (0011) or later.
- Continue Watching (0009) and TMDB/MDBList enrichment (0017).
- Trailers in the hero.
- Genre and other filter extras on Home rows, which are covered by Search/Discover (0005).

## Decisions

**1. Pure catalog core.**
- `src/data/catalog/catalog_url.{hpp,cpp}`: `buildCatalogUrl(baseUrl, type, id, skip, extras)`, ported line for line from `buildCatalogUrl`. `type` and `id` are inserted as they are, as NuvioTV does. Only extra keys and values are encoded, with `%20` for spaces.
- `src/data/catalog/catalog_parser.{hpp,cpp}`: catalog response JSON to `std::vector<MetaPreview>`, with the drop and de-duplication rules.
- `src/domain/catalog_paging.{hpp,cpp}`: `supportsExtra`, `skipStep` and `nextPage`.
- `src/ui/home/home_selection.{hpp,cpp}`: `selectHomeCatalogs(addons)`, `rowTitle(catalog)` and `pickHeroItems(rows, 7)`.

All of these are unit-tested with fixture responses in `tests/fixtures/catalog/`.

**2. `CatalogRepository::load(addon, catalog, skip, extras, cb)`.** Each request goes through `HttpClient` with permissive TLS and the default cache policy, so `Cache-Control` from add-ons is respected. Results are delivered on the UI thread as a `CatalogPage { items, rawCount, nextSkip, hasMore }`.

**3. Home view structure.**
- The Home tab is a vertical `brls::ScrollingFrame`.
- Its content `Box` holds the `HeroCarousel`, then one `CatalogRowView` per selected catalog. Each row is a header (title and caption) above a horizontal `brls::HScrollingFrame` of `PosterCell`s, plus a trailing "See all" cell when the catalog has more pages.
- Alternative: a vertical `RecyclerFrame` of rows. Rejected for now. Rows have different heights and contain horizontal scrollers, and borealis's recycler is designed for uniform cells. A typical home has 5–20 rows, so this stays manageable.
- Memory is controlled per row instead. A row more than 3 rows away from the viewport releases its poster textures (`ImageLoader::cancel` plus clearing the image) and reloads them from the memory or disk cache when it comes back.

**4. Lazy loading.**
- Rows 0–3 request immediately. Rows 4 and up start in a "pending" state with a fixed-height placeholder.
- Each frame, the scroll offset is checked with a cheap `onLayout`/scroll callback. A pending row whose top is within one row height of the visible area starts its request.
- Load-more inside a row fires when a cell within the last 5 gets focus, and the row isn't loading and has more pages.

**5. Hero.**
- `pickHeroItems` follows `slotShuffled`:
  - `slot = 7 / rows + (index < 7 % rows ? 1 : 0)`;
  - items with `background` come first, items without it only fill unused slots;
  - ids already used are skipped.
- NuvioTV sorts candidates with a stable hash key (`stableHeroSortKey`). We keep row order, which is deterministic and simpler.
- The hero recomputes only when a row's first page arrives, not on load-more, so it doesn't jump around.
- Auto-advance every 8 s while the hero isn't focused; left and right move between items when it is.

**6. Navigation.**
- See all is a new `brls::Activity` (`CatalogSeeAllActivity`) pushed on top of the shell. It holds a `RecyclerFrame` grid of `PosterCell` rows: 6 posters per row at 1280×720, with uniform heights.
- The details stub is a small activity that 0006 replaces. The selection hook is `openDetails(MetaPreview, sourceAddon)` in `src/ui/navigation.hpp`, so 0006 changes only that function.

**7. Rebuild on add-on changes.**
- Home subscribes to `AddonRepository`. On a change it recomputes the selected catalog keys (`addonId_type_catalogId`, as in NuvioTV's `catalogRowLegacyKey`).
- If the keys and order haven't changed (for example only a manifest's description changed), it does nothing. Otherwise it rebuilds the rows and keeps already-loaded pages for keys that still exist.

**8. Poster cell.**
- `PosterCell` is 126×189 with a 12 px corner radius, a 2-line title label under it, a focus scale of 1.02 and the theme's white highlight, following `ContentCard.kt` and `SizeTokens.kt`.
- `posterShape: landscape` uses 224×126 and `square` uses 160×160, as NuvioTV's `PosterShape` does.

## Risks / Trade-offs

- [Non-recycled horizontal rows with many items (some catalogs return 100 per page) cost layout time and memory] → Cap each row at 200 items. Beyond that, "See all" is the way to browse. Release textures for off-screen rows (Decision 3). Measure on hardware, and switch to a recycler per row if it's too slow.
- [Many add-ons mean many requests at once] → The worker pool limit (4) and the eager limit (4 rows) bound this.
- [Catalogs that require a non-search extra with no default (such as a required `genre`) return errors] → NuvioTV shows these and lets them fail. We do the same, so the failed row is hidden.
- [Hero quality without TMDB] → Cinemeta items include backgrounds and logos, so this is good enough until 0017.

## Known issues

- **Vertical scrolling on Home still lags on the Switch** (found in 5.4, 2026-10-30).
  - Borealis only culls leaf views. Every row and poster is a nested `Box`, so at first all posters of all rows were drawn on every frame.
  - `ViewportBox` (`src/ui/common/viewport_box.hpp`) now draws only the rows and posters that overlap the scrolling area. That improved things on hardware, but some lag remains.
  - Not yet measured. Candidates:
    - Every loaded row keeps all of its cells (up to about 100 views each) in the layout tree, so Yoga re-lays out more than the screen shows.
    - The labels are measured and drawn each frame.
    - Textures are released and reloaded when a row crosses the release distance.
    - A first page arriving and being laid out during scrolling.
  - Possible fixes: a recycler per row as planned above, fewer cells built per row until it scrolls, or tuning `RELEASE_DISTANCE`. Revisit before Home gets more rows (0009 Continue Watching) or Search reuses the rows (0005).

## Open Questions

- Should catalogs whose required extra (other than `search`) has a `default` or `options` be requested with that default instead of failing? NuvioTV doesn't, and the proposal keeps parity. Revisit if real add-ons hit this.
