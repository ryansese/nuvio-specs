## 1. Models and pure logic

- [ ] 1.1 Add `src/domain/meta_preview.hpp` (`MetaPreview`, `PosterShape`) and `catalog_row.hpp`
- [ ] 1.2 Add `catalog_url::buildCatalogUrl` with unit tests for no extras, skip only, extras with spaces, skip already present, and a base URL with a query
- [ ] 1.3 Add `catalog_parser` with unit tests for missing id or name, duplicate ids, type fallback, string `imdbRating`
- [ ] 1.4 Add `catalog_paging` (`supportsExtra`, `skipStep`, `nextPage`) with unit tests covering `pageSize`, skip options and the 100 default
- [ ] 1.5 Add `home_selection` (`selectHomeCatalogs`, `rowTitle`, `pickHeroItems`) with unit tests for search-only and `showInHome` rules, ordering, de-duplication and hero slot spread

## 2. Data layer

- [ ] 2.1 Add `CatalogRepository::load` on `HttpClient` (permissive TLS, default cache policy) returning `CatalogPage` on the UI thread
- [ ] 2.2 Add `src/ui/navigation.hpp` with `openDetails(...)` and a details stub activity (name, type, id, source add-on)

## 3. Home UI

- [ ] 3.1 Add `PosterCell` (poster/landscape/square sizes, title label, focus style, `ImageLoader` with release and reload)
- [ ] 3.2 Add `CatalogRowView` (header, horizontal scroller, trailing "See all" cell, load-more at the last 5 items, 200-item cap)
- [ ] 3.3 Add `HeroCarousel` (background, logo or name, release info, rating, description, 8 s auto-advance, left/right when focused)
- [ ] 3.4 Add `HomeTab`, replacing the 0001 placeholder: eager first 4 rows, lazy pending rows, hide failed or empty rows, release textures of distant rows
- [ ] 3.5 Add Home states: loading, "No catalog add-ons installed" with a button to Add-ons, all-failed error with Retry
- [ ] 3.6 Subscribe Home to `AddonRepository` and rebuild only when the catalog key list changes, keeping loaded pages
- [ ] 3.7 Add i18n strings (type labels, "from %s", See all, empty and error states)

## 4. See all

- [ ] 4.1 Add `CatalogSeeAllActivity` with a `RecyclerFrame` poster grid and paging
- [ ] 4.2 Restore focus to the originating row and item when returning to Home

## 5. Verification

- [ ] 5.1 `ctest --test-dir build_desktop --output-on-failure` passes
- [ ] 5.2 Desktop: Home shows Cinemeta rows and hero; scroll through rows (lazy loading visible), page a row to the end, open See all, open the stub and return with focus restored; reorder add-ons and see Home rebuild
- [ ] 5.3 Desktop: start with only OpenSubtitles enabled and see the empty state; go offline and see the error state and Retry
- [ ] 5.4 Switch: check scrolling smoothness and memory with all rows loaded on hardware or emulator; note frame drops and adjust the release distance if needed
- [ ] 5.5 CI passes on both jobs
