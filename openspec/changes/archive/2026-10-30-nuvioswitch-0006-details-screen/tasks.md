## 1. Meta model and lookup

- [x] 1.1 Add `src/meta/meta.{hpp,cpp}`: Meta and Video structs and JSON parsing (alias fields `name`/`number`/`description`, inline streams kept as raw JSON)
- [x] 1.2 Add type inference (`tv` → `series`, ID markers) and add-on matching helpers (`meta` resource type and ID-prefix checks, case-insensitive)
- [x] 1.3 Add `candidates(addons, type, id)` implementing the five-step order with de-duplication
- [x] 1.4 Add a `Cache-Control` TTL helper (default 6 h, minimum 5 min, no-cache/no-store → 5 min)
- [x] 1.5 Add `MetaRepository::lookup`: candidates in sequence, then source add-on fallback; LRU cache of 32 with TTL; shared in-flight lookups; results via `brls::sync`
- [x] 1.6 Unit tests: candidate order cases, type inference, meta URL with base query, TTL parsing, LRU eviction, default video ID and season selection helpers

## 2. Details screen

- [x] 2.1 Add `resources/xml/activity/details.xml` and `src/activity/details_activity.{hpp,cpp}` with loading, error (Retry) and loaded states
- [x] 2.2 Hero: backdrop with gradient, logo or title, release info, runtime, IMDb rating, genres, description, director and cast, Play button
- [x] 2.3 Season tabs and an episode `RecyclerFrame` with `resources/xml/views/episode_cell.xml` (thumbnail, SxEy, title, date, overview)
- [x] 2.4 Initial season selection (default video, first season above 0, first season, single virtual season)
- [x] 2.5 Wire poster selection in Home, "see all" and Search to push `DetailsActivity` with ID, type and source add-on
- [x] 2.6 Play and episode selection build the stream-picker arguments (no-op until 0007 lands)
- [x] 2.7 Add the i18n strings

## 3. Verification

- [x] 3.1 `ctest --test-dir build_desktop` passes
- [x] 3.2 Desktop: open a Cinemeta movie and series from Home and Search; check hero fields, season tabs, episode list, B returns focus to the poster, reopening uses the cache
- [x] 3.3 Desktop: with a broken add-on placed first, check the next candidate is used; with none working, check the error and Retry
- [x] 3.4 Switch: build the `.nro` and open a long series (many episodes) on hardware or in an emulator; check scrolling stays smooth and memory is stable
