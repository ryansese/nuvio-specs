## 1. Player settings

- [ ] 1.1 Add `src/core/settings/player_settings.{hpp,cpp}`: struct with NuvioTV's names and defaults (autoplay, next episode, still watching, skip, languages, subtitle style), per-profile `player_settings.json` load/save
- [ ] 1.2 Add the `NUVIO_INTRODB_API_URL` CMake cache variable and generated `build_config.hpp`
- [ ] 1.3 Unit tests: defaults, round-trip, unknown keys ignored

## 2. Autoplay selection

- [ ] 2.1 Add `isEffectivelyEnabled` and `isRegexSelectionConfigured` (`std::regex`, ECMAScript, icase; at least one letter or digit)
- [ ] 2.2 Add `selectAutoPlayStream`: add-on scope, binge-group priority and binge-group-only, First stream, Regex with exclusion words from negative lookaheads, the playable rule
- [ ] 2.3 Unit tests: first-stream skips external and unsupported streams, regex include and exclude, binge group in Manual, invalid regex
- [ ] 2.4 Add `stream_link_cache` (per profile, keyed by `videoId`/`contentId`, `cachedAtMs`, max age) and clear it when the player fails before the first frame; unit tests for expiry
- [ ] 2.5 Wire autoplay into the stream picker: select per batch, a timeout timer, a "Starting …" banner, cancel with B, reuse last link before querying add-ons

## 3. Next episode

- [ ] 3.1 Add `resolveNextEpisode` (season/episode order, absolute numbering) and `shouldShowNextEpisodeCard` (percentage 97–100, minutes 0–3.5, outro and post-credits handling), with unit tests ported from NuvioTV cases
- [ ] 3.2 Add the next-episode card overlay (thumbnail, title, Play next, B to dismiss, 10 s countdown when autoplay-next is on)
- [ ] 3.3 Add the next-episode state machine: mark the current episode completed, resolve a stream with the Manual / binge-group-only / fallback rules and timeout, fall back to the stream picker
- [ ] 3.4 Add the still-watching prompt (count of automatic episodes in a row, threshold, 60 s countdown, Continue resets, exit on timeout), with unit tests for the trigger rule

## 4. Skip segments

- [ ] 4.1 Add the IntroDB client (episode and movie queries, sec/ms fields, validity, movie credits trimmed before post-credits) and the AniSkip client for `mal:` IDs
- [ ] 4.2 Add `mergeByPriority` (opening/ending/recap categories, IntroDB then AniSkip) and a per-session cache, with unit tests
- [ ] 4.3 Add `findActiveSkipInterval`, `followingPostCreditsScene` and the button visibility rules, with unit tests
- [ ] 4.4 Add the skip button overlay (label by type, seek to end, 10 s auto-hide, shown again with the controls, B dismiss)
- [ ] 4.5 Add auto-skip for the configured types, once per interval, with a "Skipped …" notice

## 5. Languages and subtitle style

- [ ] 5.1 Add language mapping: settings code to the mpv `alang`/`slang` list (639-1 + 639-2), "Device" from `brls::Application::getLocale()`; unit tests
- [ ] 5.2 Apply forced-subtitle preference and `sub-filter-sdh` on load; keep a manual track choice for the session
- [ ] 5.3 Apply the subtitle style properties on load and when the style changes; unit tests for the value mapping

## 6. Verification

- [ ] 6.1 `ctest --test-dir build_desktop` passes with all new selection, next-episode and skip tests
- [ ] 6.2 Desktop: First stream mode starts the first playable stream; a regex with exclusions picks the expected one; a timeout with no match leaves the picker open
- [ ] 6.3 Desktop: play an episode near its end and check the card appears at the threshold; "Play next" reuses the binge group; still-watching appears after 3 automatic episodes when enabled
- [ ] 6.4 Desktop: with `NUVIO_INTRODB_API_URL` set, the skip button appears on a known title and seeks correctly; auto-skip works for a selected type
- [ ] 6.5 Switch: build the `.nro`; on hardware or an emulator check the card countdown, the skip button focus with the gamepad, and subtitle styling and language selection on a multi-track file
