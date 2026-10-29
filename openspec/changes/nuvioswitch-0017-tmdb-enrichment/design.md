## Context

NuvioTV's TMDB support has two layers:
- `TmdbService.kt` handles id mapping (IMDb ↔ TMDB, cached, with requests in flight shared).
- `TmdbMetadataService.kt` handles enrichment: details, credits, images, age rating, alternative titles and trailers fetched in parallel, cached per id and language. It also covers episode enrichment per season, "more like this" and collections.

`MetaDetailsViewModel.enrichMeta` merges the result into the add-on `Meta` group by group (lines 1528–1660). Each group is gated by a `TmdbSettingsDataStore` toggle, and the whole feature is off by default.

The Switch details screen (0006) loads meta from add-ons and renders a hero section plus seasons and episodes.

## Goals / Non-Goals

**Goals:**
- Richer details pages (logo, backdrop, cast, localised text, episode stills) with NuvioTV's exact merge rules and defaults.
- Support `tmdb:` ids from add-ons.

**Non-Goals:**
- Trailers (`fetchTmdbTrailers`, YouTube playback), "more like this", TMDB collections, production companies and networks, and cast or company browse pages.
- Enriching Home rows or Continue Watching (`tmdb_modern_home_enabled`, `tmdb_enrich_continue_watching`).
- Alternative-title matching and localisation heuristics beyond what `fetchEnrichment` does by default.

## Decisions

**1. `TmdbClient` + `TmdbIdResolver` + `TmdbEnrichmentService` in `src/core/tmdb/`.**
- The service runs the four or five calls concurrently on the 0002 worker pool and joins them.
- It keeps an LRU of 64 enrichments plus a map of in-flight promises, so revisits and duplicate requests are free.
- It is session-only with no disk cache; the 0002 HTTP cache already respects TMDB's `Cache-Control`.

**2. Merge as a pure function** `mergeEnrichment(Meta, Enrichment, TmdbSettings) -> Meta`, unit-tested group by group against cases taken from `enrichMeta`. The details view model applies it when the enrichment arrives, so the add-on meta renders first and the UI is never blocked on TMDB.

**3. Image and language choices copied from NuvioTV:**
- backdrop `w1280`, logo and cast photos `w500`, episode stills `w500`;
- `include_image_language` = `<lang>,<lang-region>,en,null`;
- the logo is picked from the image list in that preference order.

**4. Age rating region.** Use the region part of the language setting, or US when there is none, picking the matching `release_dates` or `content_ratings` entry, as `selectTvAgeRating` does.

**5. Fallback meta for `tmdb:` ids.**
- A 5-second timeout on the IMDb mapping keeps the screen responsive.
- The TMDB-only page has no videos, so for series the season list is empty. Streams still go through add-ons using the original id.

## Risks / Trade-offs

- [TMDB API key exposure in the `.nro`] → Same as NuvioTV. The key comes from build config and is never committed.
- [Extra network calls per details page on slow Wi-Fi] → Off by default, done in parallel, cached, and the add-on meta renders first.
- [Season or episode mismatches between add-on numbering and TMDB (anime, specials)] → Merge only by exact season and episode match. Unmatched episodes keep the add-on data.

## Open Questions

- Should the TMDB language follow the app's UI language by default instead of `en`? NuvioTV defaults to `en`, and this change does the same.
