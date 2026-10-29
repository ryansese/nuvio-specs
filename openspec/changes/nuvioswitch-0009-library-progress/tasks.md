## 1. Progress model and store

- [ ] 1.1 Add `src/core/progress/watch_progress.{hpp,cpp}`: `WatchProgress` and `WatchedItem` structs, JSON (de)serialisation, `progressPercentage`, `isInProgress`, `isCompleted` (2% / 90%)
- [ ] 1.2 Add key creation (`{id}` / `{id}_s{S}e{E}`), metadata-preserving upsert, and removal of the series-level key when an episode is saved
- [ ] 1.3 Add the recent/archive bucket split (200 newest, ties broken by key descending) and the merged read
- [ ] 1.4 Add `ProgressRepository`: load per profile, save, remove, mark completed (writes a watched item, removes progress), mark unwatched, observe changes
- [ ] 1.5 Unit tests for keys, thresholds, metadata merge, bucket split and the completed transition

## 2. Player integration

- [ ] 2.1 Add `ProgressTracker` with `onFirstFrame`, `onTick`, `onPause`, `onExit`, `onNaturalEnd`, applying the 90 s / 5 s / placeholder (1..120 999 ms) / below-1 s rules and the completed-this-session guard
- [ ] 2.2 Add an effective-duration helper (the larger of mpv's duration and the last known duration; 0 if the position is past it before the end)
- [ ] 2.3 Wire the tracker into the 0008 player activity; flush synchronously on exit
- [ ] 2.4 Pass the resume position as `start=+<sec>` on load; support "Start from beginning"
- [ ] 2.5 Unit tests for the tracker rules with a fake clock

## 3. Library

- [ ] 3.1 Add `src/core/library/` with a `SavedLibraryItem` struct, `LibraryRepository` (add, remove, contains, list), dedupe by `id`+`type`, and stored sort and filter settings
- [ ] 3.2 Add sort (provider order, added newest/oldest, title A–Z/Z–A, ignoring case) and type filter functions, with unit tests
- [ ] 3.3 Replace the Library placeholder tab with `library_tab.{hpp,cpp}` + XML: poster grid, sort and filter selectors, empty state, open Details on select

## 4. Continue Watching

- [ ] 4.1 Add the next-up resolver (port of `resolveNextUpVideoFromMeta` with unaired episodes off), with unit tests for same-season, rollover, missing dates and season 0
- [ ] 4.2 Add Continue Watching assembly: in-progress dedupe per series, next-up seeds with suppression, merge and sort, the `cw_hidden.json` hide list, at most 32 lookups 4 at a time; unit tests
- [ ] 4.3 Add the Continue Watching row at the top of Home: progress bar on in-progress cards, "Next up" label, hidden when empty
- [ ] 4.4 Add row actions: select opens the stream picker resuming the item; the X menu offers Start from beginning, Go to details, Remove

## 5. Details integration

- [ ] 5.1 Add the "Add to library" / "Remove from library" button
- [ ] 5.2 Add the movie "Mark as watched/unwatched" button, and episode check marks, progress bars and the episode options menu
- [ ] 5.3 Add play-button labels: "Resume · Xm left", "Play S{n}E{m}"
- [ ] 5.4 Add i18n strings for all new labels to `resources/i18n/en-US/app.json`

## 6. Verification

- [ ] 6.1 `ctest --test-dir build_desktop` passes, including all new progress, library and Continue Watching tests
- [ ] 6.2 Desktop: play a film for more than 90 s, exit, and check it appears in Continue Watching and resumes at the saved position; seek past 90% and exit, and check it's marked watched and leaves the row
- [ ] 6.3 Desktop: finish an episode and check the next episode appears as "Next up"; add and remove library items and check the sort options persist across a restart
- [ ] 6.4 Switch: build the `.nro`, repeat 6.2 on hardware or an emulator, and check that pressing + right after leaving the player keeps the progress
