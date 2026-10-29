# NuvioSwitch-0006 Details screen

## Why

Home (0004) and Search (0005) list titles, but choosing one leads nowhere. NuvioTV opens a details screen that loads the title's full metadata from the best add-on and lets the user pick a season and episode before choosing a stream. This is the step between browsing and playback (0007 and 0008).

## What Changes

- Add a details screen that opens from any poster, pushed as a borealis activity over the tab shell. B goes back to where the user came from.
- **Load metadata the way NuvioTV does by default** (`MetaDetailsViewModel.kt` with `preferExternalMetaAddonDetail = true`, `MetaRepositoryImpl.getMetaFromAllAddons`):
  - Try the meta-capable add-ons in priority order, and use the first one that returns a `meta`.
  - Only if that fails, try the add-on the poster came from.
  - Results are cached in memory (LRU of 32), with a TTL taken from `Cache-Control` (default 6 h, minimum 5 min). Two requests for the same title at the same time share one fetch.
- **Hero section:**
  - backdrop, logo (or the title as text), year/release info, runtime, IMDb rating, genres, description, cast and director;
  - a Play button.
- **Series:** season tabs plus an episode list (thumbnail, number, title, release date, overview).
  - The selected season is the `behaviorHints.defaultVideoId` episode's season, otherwise the first season above 0, otherwise specials.
  - Videos with no season numbers become a single virtual season.
- **Play** opens the stream picker (0007) with the right video ID:
  - movies use `behaviorHints.defaultVideoId` or the meta ID;
  - series use the chosen episode's video ID, and Play picks the first episode of the selected season.
- **Errors:** loading and error states, with a Retry action.

## Capabilities

### New Capabilities
- `nuvioswitch-details`: opening a title, choosing which add-on supplies its metadata, caching, the hero and episode layout, and handing a video ID to the stream picker.

### Modified Capabilities
None.

## Impact

- **Code (NuvioSwitch):**
  - `src/meta/` holds the meta model and parser (Meta, Video, behaviorHints) and `MetaRepository` (candidate ordering, cache, sharing duplicate in-flight requests);
  - `src/activity/details_activity.{hpp,cpp}`;
  - `resources/xml/activity/details.xml` and `resources/xml/views/episode_cell.xml`;
  - i18n strings.
- **Reuses from earlier changes:** the add-on model with `resources`, `types` and `idPrefixes` (0003); the catalog item navigation from Home and Search (0004, 0005); the HTTP client with response headers, the image loader for backdrops, logos and thumbnails, and storage (0002).
- **Depends on:** 0002, 0003, 0004. Search (0005) navigation hooks into it. Play hands off to 0007.
- **Out of scope:**
  - Add to library, Mark watched and the resume/"next up" episode, all in 0009;
  - trailers, cast pages, ratings from MDBList, comments, "more like this" and collections;
  - TMDB fallback and `tmdb:` ID lookup (0017).
