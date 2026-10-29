# NuvioSwitch-0009 Library, watch progress and Continue Watching

## Why

After 0008 the app can play a stream, but it forgets everything once playback stops. Without saved progress the user can't resume a film, can't see what they were watching, and can't keep a list of titles. These are the features that turn a player into a media app, and the account sync (0014) and tracker integrations (0015, 0016) later build on the same local data.

## What Changes

- **Watch progress store.**
  - One entry per movie or episode, keyed `{contentId}` for movies or `{contentId}_s{S}e{E}` for episodes.
  - Kept per profile. The 200 most recent entries go in a recent bucket; the rest are archived.
- **Progress saving during playback.**
  - Saves every 90 s if the position moved at least 5 s, and on pause, on exit and at natural end.
  - Nothing is saved before the first frame, below 1 s, or for streams of 2:01 or shorter (placeholder streams).
- **Thresholds.** From 2% an item counts as in progress. At 90% it's marked watched and its progress entry is replaced by a watched record.
- **Resume.** Playing an in-progress item starts at its saved position, unless the user picks "Start from beginning".
- **Continue Watching row** at the top of Home:
  - in-progress movies and episodes, with one in-progress entry per series (the latest);
  - "next up" episodes after the furthest completed episode of a series;
  - sorted by last watched, newest first.
- **Library.**
  - "Add to library" / "Remove from library" on the Details screen.
  - The Library tab becomes a poster grid of saved items, with sort options: provider order, date added (both directions), title A–Z and Z–A.
- **Watched state.**
  - "Mark as watched" / "Mark as unwatched" for movies and episodes on the Details screen.
  - Watched episodes get a check mark in the episode list.
- **Removing a Continue Watching item.** From the row's options menu (X / long press), "Remove" deletes it.

## Capabilities

### New Capabilities
- `nuvioswitch-library`: the saved-items library, its storage, the Library tab and its sort options.
- `nuvioswitch-watch-progress`: the progress and watched-items stores, key format, thresholds, resume rules and next-up resolution.

### Modified Capabilities
None. The extensions below are `ADDED` requirements on capabilities introduced by earlier, not-yet-archived changes:
- `nuvioswitch-home-catalogs` (0004): adds the Continue Watching row.
- `nuvioswitch-details` (0006): adds the library toggle, the watched toggle, resume and the watched marks on episodes.
- `nuvioswitch-playback` (0008): adds progress saving and resume during playback.

## Impact

- **Depends on:**
  - 0002 (storage, image loading)
  - 0004 (Home rows)
  - 0006 (Details screen, `Meta.videos`)
  - 0008 (player position and duration, pause and end events)
- **Code (`repos/NuvioSwitch`):**
  - `src/core/progress/` (store, keys, thresholds, next-up resolver)
  - `src/core/library/` (store, sort)
  - `src/tabs/library_tab.*`
  - a Continue Watching row view
  - Details and player hooks
  - unit tests for every pure rule
- **Storage:** new per-profile files `watch_progress_recent.json`, `watch_progress_archive.json`, `watched_items.json` and `library.json` under the 0002 storage root. Their fields match NuvioTV's `WatchProgress`, `WatchedItem` and `SavedLibraryItem`, so 0014 can sync them without a migration.
- **Behaviour references (NuvioTV):**
  - `domain/model/WatchProgress.kt`, `domain/model/WatchedItem.kt`, `domain/model/SavedLibraryItem.kt`
  - `data/local/WatchProgressPreferences.kt`, `data/local/WatchProgressBuckets.kt`, `data/local/LibraryPreferences.kt`
  - `ui/screens/player/PlayerRuntimeControllerPlaybackEvents.kt`
  - `ui/screens/home/HomeViewModelContinueWatching.kt`
  - `ui/screens/library/LibraryViewModel.kt`
