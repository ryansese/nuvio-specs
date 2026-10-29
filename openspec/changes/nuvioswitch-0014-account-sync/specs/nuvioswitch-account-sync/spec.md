## ADDED Requirements

### Requirement: Sync only when signed in
Sync SHALL run only while an account session exists. All RPCs SHALL carry `p_profile_id` set to the active profile's id. The one exception is add-ons: when the active profile uses the primary profile's add-ons, they SHALL be synced as profile 1.

#### Scenario: Signed out
- **WHEN** the user is signed out and adds a library item
- **THEN** no network request is made and the item stays local

#### Scenario: Profile scoping
- **WHEN** profile 2 is active and the user marks an episode watched
- **THEN** the push uses `p_profile_id: 2`

### Requirement: Stable origin client id
The app SHALL generate a client id once, made of `nuvio-switch-` followed by 32 characters from `[a-z0-9]`, store it globally, and send it as `p_origin_client_id` on every push and delete RPC, as `core/sync/SyncClientIdentity.kt` does.

#### Scenario: Same id across launches
- **WHEN** the app pushes progress, restarts, and pushes again
- **THEN** both requests carry the same `p_origin_client_id`

### Requirement: Add-on sync
After a local add-on change, the app SHALL push the full ordered list with `rpc/sync_push_addons {p_addons: [{url, sort_order, enabled, name?}], p_profile_id, p_origin_client_id}`. `sort_order` is the list index and `name` is the user-set name if there is one. To pull, the app SHALL read `GET /rest/v1/addons?user_id=eq.<effective user id>&profile_id=eq.<id>`. It SHALL replace the local list in `sort_order` order, keeping the custom names and enabled flags, and then refresh manifests.

#### Scenario: Add-on added on another device
- **WHEN** the user installs an add-on on NuvioTV and the Switch pulls
- **THEN** the add-on appears on the Switch in the same position

#### Scenario: Reorder pushed
- **WHEN** the user moves an add-on up on the Switch
- **THEN** `sync_push_addons` is sent with the new `sort_order` values

### Requirement: Library sync
Library changes SHALL be pushed in batches of up to 500:
- upserts: `rpc/sync_push_library_items {p_items, p_profile_id, p_origin_client_id}`, where each item has `content_id`, `content_type`, `name`, `poster`, `poster_shape`, `background`, `description`, `release_info`, `imdb_rating`, `genres`, `addon_base_url`, `logo`, `added_at`;
- deletes: `rpc/sync_delete_library_items {p_keys: [{content_id, content_type}], ...}`.

Pulling SHALL work as follows:
- With no cursor stored, take a full snapshot with `rpc/sync_pull_library {p_profile_id, p_limit: 500, p_offset}`, paging until a short page, and then store the cursor from `rpc/sync_get_library_delta_cursor`.
- With a cursor, apply events from `rpc/sync_pull_library_delta {p_profile_id, p_since_event_id, p_limit: 500}` in `event_id` order, by their `operation` (upsert or delete), and advance the cursor.

#### Scenario: First sync of an existing library
- **WHEN** a user with 1,200 library items signs in on the Switch
- **THEN** the app fetches three snapshot pages, shows all 1,200 items and stores a delta cursor

#### Scenario: Incremental removal
- **WHEN** an item is removed on NuvioTV and the Switch runs a delta pull
- **THEN** the delete event removes it from the Switch library

### Requirement: Watch progress sync
Progress SHALL be pushed with `rpc/sync_push_watch_progress {p_entries: [{content_id, content_type, video_id, season?, episode?, position, duration, last_watched, progress_key}], p_profile_id, p_origin_client_id}`. `progress_key` uses the 0009 key format (`{id}_s{S}e{E}`, or the content id for movies). Positions and durations are in milliseconds and `last_watched` is in epoch milliseconds. Deletes SHALL use `rpc/sync_delete_watch_progress {p_keys: [progress_key...], ...}`. Pulls SHALL use the snapshot `rpc/sync_pull_watch_progress {p_profile_id}`, then deltas with `rpc/sync_pull_watch_progress_delta {p_profile_id, p_since_event_id, p_limit: 900}` and `rpc/sync_get_watch_progress_delta_cursor`. When the same key exists both locally and remotely, the entry with the newer `last_watched` SHALL win.

#### Scenario: Resume on Switch from TV
- **WHEN** the user stops an episode at 23 minutes on NuvioTV and the Switch pulls
- **THEN** the Switch's Continue Watching shows the episode, and playing it resumes at 23 minutes

#### Scenario: Older remote entry ignored
- **WHEN** a pulled entry has an older `last_watched` than the local entry with the same key
- **THEN** the local entry is kept, and it is pushed if it isn't yet acknowledged

### Requirement: Watched items sync
Watched marks SHALL be pushed with `rpc/sync_push_watched_items {p_items: [{content_id, content_type, title, season, episode, watched_at}], p_profile_id, p_origin_client_id}`, with `season` and `episode` set to null for movies, and deleted with `rpc/sync_delete_watched_items {p_keys: [{content_id, season?, episode?}], ...}`. Pulls SHALL use the paged snapshot `rpc/sync_pull_watched_items {p_profile_id, p_page, p_page_size: 900}`, then deltas with `rpc/sync_pull_watched_items_delta` (limit 900) and `rpc/sync_get_watched_items_delta_cursor`.

#### Scenario: Episode watched on Switch
- **WHEN** an episode passes the 90% watched threshold on the Switch
- **THEN** a watched item for that season and episode is pushed and shows as watched on NuvioTV after it syncs

### Requirement: Durable pending mutations
Every local upsert or delete of the synced kinds SHALL be recorded in a per-profile pending queue before the network call, as `core/sync/WatchStateMutationStore.kt` does. Entries SHALL be removed only after the RPC succeeds. Pulled remote data SHALL NOT overwrite a key with a pending local mutation. Deletes SHALL be flushed before upserts.

#### Scenario: Offline watching
- **WHEN** the user watches two episodes offline and later goes online
- **THEN** both progress entries and watched marks are pushed on the next sync, and the queue is empty afterwards

#### Scenario: Crash before acknowledgement
- **WHEN** the app quits after recording a mutation but before the RPC completes
- **THEN** the mutation is still pending at the next launch and is pushed then

### Requirement: Sync scheduling
The app SHALL:
- run a full pull followed by a flush after sign-in, after switching profile, and at launch while signed in;
- run a delta pull 2.5 s after the app returns to the foreground, at most once every 2 minutes;
- run a delta pull every 15 minutes while open;
- flush the queue after playback stops and after library, watched or add-on changes.

Only one sync SHALL run at a time. A forced Sync now SHALL be allowed at most once every 30 seconds. Intervals follow `core/sync/StartupSyncService.kt`.

#### Scenario: Return from HOME menu
- **WHEN** the user leaves NuvioSwitch for the HOME menu and returns 5 minutes later
- **THEN** a delta pull runs about 2.5 s after returning

#### Scenario: Sync now
- **WHEN** the user taps Sync now in Settings → Account twice within 10 seconds
- **THEN** only one sync runs, and the section shows the new last-synced time

### Requirement: Sync failures don't block the app
A failed sync SHALL be logged and retried at the next trigger. It SHALL NOT show modal errors or block browsing or playback. An expired session SHALL go through the 0013 refresh. An invalid session SHALL stop sync until the user signs in again.

#### Scenario: Backend unreachable
- **WHEN** the Supabase host can't be reached during a scheduled pull
- **THEN** the app keeps working with local data, and Settings shows the last successful sync time
