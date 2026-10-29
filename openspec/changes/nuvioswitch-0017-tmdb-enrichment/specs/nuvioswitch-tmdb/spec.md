## ADDED Requirements

### Requirement: TMDB client
The app SHALL call TMDB at `https://api.themoviedb.org/3/` with `api_key=<TMDB_API_KEY>` on every request, and build image URLs as `https://image.tmdb.org/t/p/{size}{path}`. TMDB SHALL be unavailable when `TMDB_API_KEY` is empty.

#### Scenario: No key
- **WHEN** the build has no TMDB key
- **THEN** no TMDB request is ever made and the TMDB Settings section is hidden

### Requirement: TMDB id resolution
The app SHALL resolve a TMDB id from a content id as `TmdbService.ensureTmdbId` does:
1. Strip the `tmdb:`, `movie:` and `series:` prefixes, and take the part before the first `:` or `/`.
2. If it starts with `tt`, look it up with `GET find/{imdb_id}?external_source=imdb_id` and take `movie_results` or `tv_results` according to the type.
3. If it is all digits, use it as is.
4. Otherwise use the meta's IMDb id, if there is one.

IMDb-to-TMDB results SHALL be cached by `{imdb id}:{type}` for the session, and identical lookups already in flight SHALL be shared.

#### Scenario: IMDb series episode id
- **WHEN** the content id is `tt0903747:1:1` and the type is series
- **THEN** the app requests `find/tt0903747` and uses the first `tv_results` id

#### Scenario: Unresolvable id
- **WHEN** the content id is `kitsu:1` and the meta has no IMDb id
- **THEN** no enrichment happens and the details screen shows add-on data only

### Requirement: Enrichment fetch
For a resolved TMDB id, the app SHALL fetch these in parallel, using the configured language:
- details: `movie/{id}` or `tv/{id}`;
- credits: `movie/{id}/credits`, or `tv/{id}/aggregate_credits`;
- images: `{type}/{id}/images` with `include_image_language=<lang>,<lang-region>,en,null`;
- age rating: `movie/{id}/release_dates` or `tv/{id}/content_ratings`, choosing the user's region.

For series with episode enrichment on, it SHALL also fetch `tv/{id}/season/{n}` for each season the meta lists. Results SHALL be cached in memory by `{tmdbId}:{type}:{language}`, and identical requests in flight SHALL be shared. The produced enrichment holds:
- localised title and description;
- genres and TMDB rating;
- runtime, status, age rating, countries and original language;
- backdrop (`w1280`) and logo (`w500`);
- director and writer;
- cast with names, characters and photos (`w500`);
- per-episode title, overview, still (`w500`) and runtime.

#### Scenario: Cached on revisit
- **WHEN** the user opens the same movie's details twice in a session
- **THEN** TMDB is called only the first time

#### Scenario: TMDB down
- **WHEN** TMDB requests fail or time out
- **THEN** the enrichment is empty and the error is only logged
