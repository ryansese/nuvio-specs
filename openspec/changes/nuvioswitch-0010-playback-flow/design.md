## Context

By 0009 the app has the stream picker (0007), the mpv player (0008), progress tracking and natural-end events (0009). NuvioTV's playback flow is spread across many files. All paths below are under `app/src/main/java/com/nuvio/tv/`:

- **Stream selection:** `core/player/StreamAutoPlaySelector.kt`, a pure function, and `core/player/StreamAutoPlayPolicy.kt` (`isEffectivelyEnabled`, `isRegexSelectionConfigured`).
- **Settings and defaults:** `data/local/PlayerSettingsDataStore.kt` (`PlayerSettings`, `SubtitleStyleSettings`, `AutoSkipSegmentType`).
- **Last-link cache:** `data/local/StreamLinkCacheDataStore.kt`, a JSON payload per content key with `cachedAtMs`.
- **Next episode:**
  - `ui/screens/player/PlayerNextEpisodeRules.kt`: `resolveNextEpisode`, `shouldShowNextEpisodeCard`, `OUTRO_SEGMENT_TYPES`, `POST_OUTRO_AUTOPLAY_GAP_MS = 5000`;
  - `ui/screens/player/PlayerRuntimeControllerStreams.kt`: `playNextEpisode`, with the manual and binge-group-only branches and the timeout wait.
- **Skip:**
  - `data/repository/SkipIntroRepository.kt`: `toSkipIntervals`, `mergeByPriority`, `segmentCategory`;
  - `ui/screens/player/SkipIntroVisibilityRules.kt`: `findActiveSkipInterval`, `followingPostCreditsScene`, `isSkipIntroButtonVisible`, 10 s auto-hide.
- **Still watching:** `ui/screens/player/PlayerRuntimeControllerStillWatching.kt` (60 s countdown, default threshold 3).

## Goals / Non-Goals

**Goals:**
- Port the selection, next-episode and skip rules as pure, unit-tested C++ that behaves like NuvioTV.
- Keep the settings model compatible with NuvioTV's names and defaults, so 0014 can sync them later.

**Non-Goals:**
- Plugin streams and debrid-specific ordering. The "plugins only" source choice is dropped; debrid comes in 0018.
- The Anime-Skip GraphQL provider, which needs a user client ID. Mapping IMDb or Kitsu IDs to MAL needs Simkl's resolver (0016). Until then AniSkip runs only for `mal:` IDs.
- Post-play recommendations for movies, the parental guide overlay and the pause overlay.
- The settings UI. It belongs to 0011; here settings are read from `player_settings.json` with defaults.

## Decisions

**1. Pure modules with thin UI glue.**
- `autoplay_selector` holds `orderAddonStreams` (from 0007, reused), `isEffectivelyEnabled` and `selectAutoPlayStream`.
- `next_episode_rules` holds `resolveNextEpisode` and `shouldShowNextEpisodeCard`.
- `skip_rules` holds `toSkipIntervals`, `mergeByPriority`, `findActiveSkipInterval`, `followingPostCreditsScene` and `isSkipButtonVisible`.
- They take plain structs and are covered by doctest. NuvioTV's own test cases (for example in `app/src/test/`) are good sources of fixtures.

**2. Regex engine: `std::regex` with ECMAScript and `icase`.**
- NuvioTV uses Java regex. ECMAScript supports the negative lookahead that users write in their patterns, which is the common case.
- Exclusion words are extracted with the same pattern NuvioTV uses (`\(\?![^)]*?\(([^)]+)\)`) and matched as whole words, ignoring case.
- Invalid patterns are caught (`std::regex_error`) and count as "not configured".
- Alternative: PCRE2 from portlibs. Rejected for now; it adds a dependency for little gain. Revisit if users report Java-specific patterns failing.

**3. Autoplay runs inside the stream picker activity.**
- The picker (0007) already receives results per add-on.
- Selection runs after each batch, and a borealis timer enforces the timeout.
- A "Starting <stream> in …" banner shows the choice. B cancels.
- The player is launched exactly as a manual choice would launch it.

**4. Next-episode flow is a small state machine owned by the player:** Idle → CardShown → Countdown → Resolving → Launching, plus StillWatching.
- **Resolving** reuses the picker's query code without showing the picker, for the same timeout.
- When it fails, it pushes the stream picker for the next episode.
- The current episode is marked completed through 0009's tracker before switching.

**5. Skip data is fetched at playback start on a worker; the player polls the active interval on its existing position tick.**
- Auto-skip remembers which intervals it has already skipped (by start time), so a user who seeks back isn't thrown forward again.

**6. Language and style mapping.**
- Language preferences use ISO 639-1 codes in settings. They're expanded to `639-1,639-2` pairs for mpv `alang`/`slang` (for example `ja,jpn`), because files tag tracks either way.
- "Device" uses the system language from borealis (`brls::Application::getLocale()`), mapped to its language part.
- Style: `sub-scale = size/100`, `sub-pos = 100 - verticalOffset`, colours written as `#AARRGGBB` strings, `sub-outline-size = outlineWidth`, `sub-bold`.
- These apply to text subtitles only, because 0008 sets `sub-ass-override=no`.

**7. IntroDB URL is a build-time value.**
- It's a CMake cache variable `NUVIO_INTRODB_API_URL`, default empty, written into a generated `build_config.hpp` alongside the version header from 0001.
- 0013 adds the gitignored `config.local.json` for secrets, which can then supply this value too.

## Risks / Trade-offs

- [`std::regex` differs from Java regex for some constructs (possessive quantifiers, inline flags)] → Treat a failure to compile as "not configured" and fall back to Manual, which is safe. Note it in the setting's help text in 0011.
- [`std::regex` on large stream lists is slow on the Switch CPU] → Stream lists are small (tens to a few hundred items) and matching runs off the UI thread.
- [The IntroDB URL is unknown, since NuvioTV keeps it in untracked `local.properties`] → It's configurable; without it only AniSkip works. See the open question.
- [A remembered link may have expired on the server (debrid or signed URLs)] → A 1-hour window, as NuvioTV uses. If the player fails before the first frame, the cached link is cleared and the picker opens.

## Open Questions

- What is the IntroDB base URL NuvioTV uses in production (`INTRODB_API_URL`)? The user needs to supply it, or IntroDB stays disabled.
