## Context

0008 gives a working mpv player with position and duration reporting, and 0002 gives per-profile JSON stores with atomic writes. NuvioTV keeps progress in DataStore as JSON maps. All paths below are under `app/src/main/java/com/nuvio/tv/`:

- `data/local/WatchProgressPreferences.kt`: the `createKey`, `upsertProgressEntries` and `mergeDisplayMetadata` functions.
- `data/local/WatchProgressBuckets.kt`: `splitWatchProgressEntries`, which caps the recent bucket at 200.
- `data/local/WatchedItemsPreferences.kt`, `data/local/LibraryPreferences.kt`.
- The player writes progress from `ui/screens/player/PlayerRuntimeControllerPlaybackEvents.kt`:
  - `saveWatchProgressIfNeeded`, `saveWatchProgress`, `saveWatchProgressInternal`;
  - `isShortPlaceholderDuration`, true for 1..120 999 ms;
  - `WATCH_PROGRESS_SAVE_INTERVAL_MS = 90_000` and `saveThresholdMs = 5000` (set in `PlayerRuntimeController.kt`).
- Continue Watching is built in `ui/screens/home/HomeViewModelContinueWatching.kt`:
  - `deduplicateInProgress`, `shouldTreatAsInProgressForContinueWatching`, `choosePreferredNextUpSeed`, `resolveNextUpVideoFromMeta`, `sortContinueWatchingItems`.
  - It also has TMDB enrichment, Trakt/Simkl sources, caches and badges, which aren't needed here.

## Goals / Non-Goals

**Goals:**
- Local, per-profile progress, watched and library stores that use NuvioTV's field names. 0014 can then push them through `sync_push_watch_progress`, `sync_push_watched_items` and `sync_push_library_items` without reshaping the data.
- The same thresholds and key rules as NuvioTV, so progress synced between devices later means the same thing.
- A simple, correct Continue Watching row and Library tab.

**Non-Goals:**
- Remote sync, Trakt/Simkl/MDBList progress sources, and their completion overrides (Simkl uses 80%). These come in 0014–0016.
- TMDB enrichment of Continue Watching, badges, "upcoming" release alerts, the `STREAMING_STYLE`/`SPLIT_UPCOMING` sort modes, and unaired next-up episodes.
- The Library's Trakt/Simkl/MDBList/Cloud source tabs, genre/year filters and custom lists.
- Bulk marking (season, or all previous episodes). Single items only for now.

## Decisions

**1. Pure logic in `src/core/progress/` and `src/core/library/`, with no borealis dependency.**
- Key creation, bucket splitting, thresholds, metadata merge, the next-up resolver, Continue Watching assembly and library sorting are all plain functions over structs.
- They're unit-tested in `NuvioSwitchTests` (0001's desktop-only doctest target).
- UI code only calls the repositories.

**2. Storage files are JSON maps or arrays mirroring NuvioTV's shapes.**
- `watch_progress_recent.json` and `watch_progress_archive.json` hold `{ key: WatchProgress }`.
- `watched_items.json` holds `[WatchedItem]`, and `library.json` holds `[SavedLibraryItem]` plus the sort and filter settings.
- Times are epoch ms and positions are ms, as in NuvioTV.
- Every write goes through 0002's write-temp-then-rename helper.
- Alternative: SQLite. Rejected; it adds a portlib and is more than a few hundred items need. The bucket split keeps each file small.

**3. In-memory cache with write-behind.**
- Repositories load once per profile and keep the maps in memory.
- Periodic saves during playback are written straight away. They're at most one every 90 s, and the SD card handles that easily.
- The write runs on 0002's worker, so playback never blocks on SD I/O.
- Exiting the player flushes synchronously before the activity is popped, so pressing + right after doesn't lose the last save.

**4. Player hook point.**
- A `ProgressTracker` object owned by the player activity.
- The player calls `onFirstFrame()`, `onTick(position, duration)` from its existing position poll, `onPause()`, `onExit()` and `onNaturalEnd()`.
- The tracker applies the 90 s / 5 s / placeholder / 1 s rules and the "already marked completed this session" guard (`hasMarkedCurrentEpisodeCompleted` in NuvioTV).
- Effective duration, as NuvioTV's `getEffectiveDuration` computes it: the larger of mpv's `duration` and the last known duration. It's treated as 0 if the position is past it and playback hasn't ended.

**5. Continue Watching assembly (trimmed port of `loadContinueWatchingPipeline`).**
1. Take all progress entries.
2. In-progress items are entries at 2% or more and below 90%. For series, keep only the latest per `contentId`.
3. Next-up seeds: for each series, take its furthest completed episode by season, then episode, then `lastWatched`. Suppress it if an in-progress episode of that series was watched after the seed completed.
4. Resolve next-up episodes from `Meta.videos`, fetched through 0006's meta repository and cache. Use `resolveNextUpVideoFromMeta`'s rules with `showUnairedNextUp = false`.
5. Merge and sort by `lastWatched` (for next-up items, the seed's `watchedAt`).

- At most 32 next-up meta lookups are made (`CW_MAX_NEXT_UP_LOOKUPS`), 4 at a time. The row shows in-progress items first and adds next-up items as their meta arrives.
- The "Remove" option on a next-up item needs a way to hide that series. It's stored as `cw_hidden.json` (`contentId` → hidden-at timestamp). The series reappears once its newest progress is later than that timestamp.

**6. Library grid.**
- A borealis `RecyclerFrame` grid of 126×189 posters, reusing 0004's poster cell. Sort and filter use `SelectorCell` dropdowns in a header.
- Provider order means insertion order, matching NuvioTV's `LibrarySortOption.DEFAULT`.

## Risks / Trade-offs

- [Meta lookups for next-up hit add-ons on every Home load] → Reuse 0006's meta LRU/TTL cache, and cap the lookups at 32.
- [mpv `duration` is 0 or unknown for some live or HLS streams] → Effective-duration rule. When duration is unknown, NuvioTV saves `progressPercent = 5`. We save position only, and the item doesn't count as in progress. That's acceptable for v0.1, and recorded as a known difference.
- [Clock skew on the Switch (an unset RTC) makes `lastWatched` unreliable] → Only relative ordering matters locally. Sync (0014) will need server timestamps.
- [Data loss if the app is killed during a write] → Write-temp-then-rename keeps the old file intact.

## Open Questions

- Should a watched movie stay visible in Continue Watching until the user removes it? NuvioTV drops it at 90%, and this spec follows NuvioTV.
