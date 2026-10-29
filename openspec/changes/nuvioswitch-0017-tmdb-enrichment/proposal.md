# NuvioSwitch-0017 TMDB enrichment

## Why

Add-on metadata varies a lot. Cinemeta has no clear logos, localised titles or episode stills, and smaller add-ons often lack cast, runtime or age ratings. NuvioTV can fill these gaps from TMDB, and also handles `tmdb:` content ids that some add-ons return. The Switch details screen from 0006 shows only what the add-on gives, so it looks sparse next to NuvioTV.

## What Changes

- **TMDB client.** Uses `https://api.themoviedb.org/3/` with the build-time `TMDB_API_KEY` as the `api_key` query parameter. Images come from `https://image.tmdb.org/t/p/{size}{path}`.
- **ID resolution.** Follows `core/tmdb/TmdbService.kt` `ensureTmdbId`:
  - `tmdb:<n>` or a numeric id is used as is;
  - `tt…` is converted with `GET find/{imdb_id}?external_source=imdb_id`, cached;
  - otherwise the meta's IMDb id is used as a fallback.
- **Details enrichment.** Follows `MetaDetailsViewModel.enrichMeta` and `core/tmdb/TmdbMetadataService.kt` `fetchEnrichment`. Details, credits (aggregate credits for TV), images and age rating are fetched in parallel in the chosen language, and merged per settings group:
  - **Artwork:** backdrop `w1280`, logo `w500` in the preferred language;
  - **Basic info:** localised title, description, genres, TMDB rating;
  - **Details:** runtime, status, age rating, country, original language;
  - **Credits:** director, writer, cast with photos `w500`;
  - **Episodes:** per-season episode titles, overviews, stills `w500` and runtimes from `GET tv/{id}/season/{n}`.
- **`tmdb:` ids.**
  - The details screen first tries to map a `tmdb:` id to IMDb (`GET {movie|tv}/{id}/external_ids`, 5 s timeout) for the add-on lookup.
  - If no add-on returns meta, a details page is built from TMDB alone (`tryApplyTmdbFallbackMeta`).
- **Settings.** A TMDB section:
  - Enable (off by default, as in NuvioTV);
  - language (default `en`);
  - toggles for Artwork, Basic info, Details, Credits and Episodes (each on by default).

## Capabilities

### New Capabilities
- `nuvioswitch-tmdb`: the TMDB client, id resolution, enrichment fetch and caching.

### Modified Capabilities
None. These are added as `ADDED` delta specs on capabilities owned by earlier changes:
- `nuvioswitch-details`: merging TMDB data into the details screen, and handling `tmdb:` ids.
- `nuvioswitch-settings`: the TMDB section.

## Impact

- **Depends on:** 0002 (HTTP, image loading), 0006 (details screen and meta resolution), 0011 (Settings), 0013 (build-time `TMDB_API_KEY`).
- **Code:** new `src/core/tmdb/` (client, id resolver, enrichment service with an in-memory LRU and in-flight de-duplication), a merge step in the details view model, a cast row, and a Settings section.
- **Config:** TMDB is unavailable when `TMDB_API_KEY` is empty, and the Settings section is hidden.
- **NuvioTV references:**
  - `core/tmdb/TmdbService.kt`, `core/tmdb/TmdbMetadataService.kt`
  - `data/remote/api/TmdbApi.kt`, `data/local/TmdbSettingsDataStore.kt`
  - `ui/screens/detail/MetaDetailsViewModel.kt` (`enrichMeta`, `tryApplyTmdbFallbackMeta`, `resolveMetaLookupId`)
