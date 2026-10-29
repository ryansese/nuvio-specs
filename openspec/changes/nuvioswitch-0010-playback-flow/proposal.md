# NuvioSwitch-0010 Playback flow: autoplay, next episode, skip intro

## Why

After 0009 the user can play and resume, but every episode still takes the same manual steps: open the stream list, pick a source, sit through the intro, then go back and choose the next episode. NuvioTV removes those steps with stream autoplay, binge-group reuse, a next-episode prompt and skip-intro buttons. Without them, watching a series on the Switch is noticeably worse than on the other Nuvio clients. This change completes the v0.1 milestone.

## What Changes

- **Stream autoplay modes.** Manual (default), First stream, and Regex match. Regex matching treats words in negative lookaheads as exclusions. Autoplay can be limited to selected add-ons.
- **Autoplay timing.** When autoplay applies, the stream picker waits up to a timeout (default 10 s) for a qualifying stream and starts it as soon as one arrives. If none qualifies, the picker stays open.
- **Reuse last link.** A stream that played successfully is remembered per content key for 1 hour. Opening the same item again within that window plays it straight away.
- **Binge-group reuse.** For the next episode, a stream with the same `behaviorHints.bingeGroup` as the current one is preferred, even in Manual mode when that option is on.
- **Next-episode prompt.**
  - A card with the next episode's title and thumbnail appears near the end. The default threshold is 99%; it can also be set as N minutes before the end, and it's adjusted to known outro or credits segments.
  - "Play next" starts the next episode through the autoplay rules above.
  - With "Autoplay next episode" on, a countdown starts it automatically.
- **"Still watching?" prompt.** Optional and off by default. It appears after 3 automatically played episodes in a row, with a 60 s countdown that stops playback.
- **Skip segments.**
  - Intro, recap, outro and movie-credits intervals are fetched from IntroDB (IMDb IDs) and AniSkip (`mal:` IDs).
  - A "Skip intro"/"Skip recap"/"Skip credits" button appears while the position is inside an interval and hides itself after 10 s.
  - Optionally, chosen segment types are skipped automatically.
  - Post-credits scenes are never skipped.
- **Preferred audio and subtitle languages.** Preferred and secondary languages, forced subtitles, and stripping SDH (hearing-impaired annotations) are applied through mpv's `alang`, `slang`, `subs-with-matching-audio` and `sub-filter-sdh` options.
- **Subtitle styling.** Size, vertical position, text colour, background colour, outline colour and width, and bold are applied through mpv `sub-*` properties.
- **Settings storage.** All of these settings are stored in a per-profile `player_settings.json` with NuvioTV's defaults. The screens to change them come in 0011; until then the defaults apply.

## Capabilities

### New Capabilities
- `nuvioswitch-autoplay`: autoplay modes, stream selection rules, timeout, last-link reuse, binge-group reuse, the next-episode prompt and "still watching".
- `nuvioswitch-skip-segments`: fetching, merging and showing skip intervals, and auto-skip.

### Modified Capabilities
None. `nuvioswitch-playback` (0008, not yet archived) gets `ADDED` requirements for language preferences and subtitle styling.

## Impact

- **Depends on:**
  - 0003 (installed add-on order and names)
  - 0006 (`Meta.videos` for the next episode)
  - 0007 (stream picker and streams arriving per add-on)
  - 0008 (player, mpv property access)
  - 0009 (progress tracker, natural-end event, watched marking)
- **Code (`repos/NuvioSwitch`):**
  - `src/core/player/autoplay_selector.*`
  - `src/core/player/next_episode_rules.*`
  - `src/core/player/stream_link_cache.*`
  - `src/core/skip/` (IntroDB and AniSkip clients, merge rules)
  - `src/core/settings/player_settings.*`
  - player overlay views (next-episode card, skip button, still-watching dialog)
  - unit tests
- **External services:**
  - IntroDB. Its base URL is a build-time setting, `NUVIO_INTRODB_API_URL` (CMake cache variable); NuvioTV reads the same key into `BuildConfig`. Empty disables IntroDB.
  - AniSkip at `https://api.aniskip.com/v2/`.
- **Behaviour references (NuvioTV):**
  - `core/player/StreamAutoPlaySelector.kt`, `core/player/StreamAutoPlayPolicy.kt`
  - `data/local/PlayerSettingsDataStore.kt`, `data/local/StreamLinkCacheDataStore.kt`
  - `ui/screens/player/PlayerNextEpisodeRules.kt`, `ui/screens/player/SkipIntroVisibilityRules.kt`, `ui/screens/player/PlayerRuntimeControllerStillWatching.kt`, `ui/screens/player/PlayerRuntimeControllerStreams.kt` (`playNextEpisode`)
  - `data/repository/SkipIntroRepository.kt`, `data/remote/api/SkipIntroApi.kt`
