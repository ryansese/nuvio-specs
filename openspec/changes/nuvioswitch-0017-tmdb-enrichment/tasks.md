## 1. TMDB core

- [ ] 1.1 Add `TmdbClient` (base URL, `api_key`, image URL builder) and the DTO parsing for details, credits, aggregate credits, images, release dates, content ratings, season details, find and external ids
- [ ] 1.2 Add `TmdbIdResolver` (`ensureTmdbId` rules, IMDb↔TMDB cache, shared in-flight requests) with unit tests
- [ ] 1.3 Add `TmdbEnrichmentService` (parallel fetch, logo and language selection, age rating by region, LRU 64, shared in-flight requests)
- [ ] 1.4 Add episode enrichment per season

## 2. Details integration

- [ ] 2.1 Add the pure `mergeEnrichment(meta, enrichment, settings)` with unit tests for each group and toggle
- [ ] 2.2 Apply the enrichment in the details view model after the add-on meta renders; show the TMDB rating label and a cast row with photos
- [ ] 2.3 Map `tmdb:` ids to IMDb with a 5 s timeout before the add-on lookup, and add the TMDB-only fallback page

## 3. Settings

- [ ] 3.1 Add the per-profile `tmdb_settings` store with NuvioTV defaults
- [ ] 3.2 Add the Settings → TMDB section (enable, language picker, group toggles disabled while off), hidden without an API key

## 4. Verification

- [ ] 4.1 Desktop with a TMDB key: enable it and check that a Cinemeta movie shows the logo, cast and TMDB rating, and that a series shows episode stills
- [ ] 4.2 Desktop: switch the language to `it-IT` and check the titles and descriptions change; turn off Artwork and check only the artwork reverts
- [ ] 4.3 Open an item with a `tmdb:` id from an add-on and check it maps to IMDb; open an unknown one and check the TMDB-only page
- [ ] 4.4 `ctest --test-dir build_desktop` passes (id resolution, merge rules, logo selection)
- [ ] 4.5 Switch `.nro` builds; on hardware, check details pages stay responsive while enrichment loads
