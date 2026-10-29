## Context

NuvioTV's details flow:

- **`ui/screens/detail/MetaDetailsViewModel.kt`:** loads the meta (lines ~750–830) and picks the season (~930–970, `findPreferredDefaultEpisode`, `getEpisodesForSeason`).
- **`data/repository/MetaRepositoryImpl.kt`:**
  - `getMetaFromAllAddons` sets the candidate order;
  - `supportsMetaType` and `supportsMetaId` do the matching, and `inferCanonicalType` works out the type;
  - `parseMaxAgeMs` sets the TTL: default 6 h, minimum 5 min;
  - LRU caches hold 32 entries, and duplicate in-flight requests share one Deferred.
- **The default setting:** `LayoutPreferenceDataStore.preferExternalMetaAddonDetail` is `true` by default. So the "all add-ons" lookup comes first, and the poster's source add-on is only a fallback.

Several screen sections (`HeroSection.kt`, `EpisodesSection.kt`, cast, trailers, comments, ratings) depend on TMDB, MDBList, Trakt or trailers. Those are later specs or out of scope.

## Goals / Non-Goals

**Goals:**
- Find metadata the same way NuvioTV does, so the same add-ons answer on both clients.
- Show the hero, seasons and episodes clearly on 720p handheld and 1080p docked.
- A clean hand-off to 0007 carrying everything the stream picker and player need.

**Non-Goals:**
- Library and watched toggles, progress bars and "next up" (0009).
- Trailers, cast detail, ratings, comments, "more like this", collections and the TMDB fallback.
- The `preferExternalMetaAddonDetail = false` mode. Only the default path is ported; the setting can come with 0011 if it's wanted.

## Decisions

**1. `MetaRepository` in `src/meta/` with no UI dependencies.**
- `candidates(addons, type, id)` is a pure function, unit-tested against the ordering rules.
- `lookup(type, id, sourceBaseUrl, callback)` runs the candidates one at a time on a worker thread (each request blocks inside 0002's HTTP pool), then falls back to the source add-on. It reports back through `brls::sync`.
- The LRU and TTL cache and the in-flight map (`std::unordered_map<key, std::vector<callback>>`) live in the repository, which is a singleton per profile. It's guarded by a mutex because lookups finish on worker threads.
- Alternative: query candidates in parallel and take the first success. Rejected; NuvioTV goes in order on purpose, because the order is the user's priority.

**2. Parse `Cache-Control` in a small pure helper** (`max-age`, `no-cache`, `no-store`). The networking layer from 0002 must return response headers, which this change relies on.

**3. Meta model.**
- `Meta` contains id, type, name, poster, background, logo, description, releaseInfo, imdbRating, runtime, genres, director[], cast[], videos[] and behaviorHints.defaultVideoId.
- `Video` contains id, title (also read from `name`), released, thumbnail, season, episode (also read from `number`), overview (also read from `description`), available and inline `streams[]`.
- Inline streams are kept as raw JSON for 0007 to parse. This mirrors `data/remote/dto/MetaResponseDto.kt`.

**4. The details screen is its own activity.**
- `brls::Application::pushActivity(new DetailsActivity(...))` keeps the tab shell's state. B pops it, and borealis restores focus to the poster that opened it.
- Layout: a full-width backdrop `Image` dimmed with a gradient rectangle. On the left, a column with the logo, metadata line, description and Play button. Below, season tabs (a horizontal box of buttons) and an episode `RecyclerFrame` (vertical list, one cell per episode).

**5. The video ID for Play** follows NuvioTV's rule: movies use `defaultVideoId` or the meta ID; series use the first episode of the selected season. A pure helper is unit-tested. Resuming at a "next up" episode is added by 0009.

## Risks / Trade-offs

- [Running candidates in order can be slow when the top candidates time out] → The same as NuvioTV. The 30 s connect timeout from 0002 applies, and a loading spinner is shown. A shorter per-candidate timeout can come later if it's a problem in practice.
- [Large `videos` arrays (long anime) are heavy to parse and render on Switch] → Parse once, group by season once, and use `RecyclerFrame` so only visible episode cells exist.
- [Backdrop images are large] → Use 0002's image loader and scale to screen size before upload, keeping memory within Switch limits.
