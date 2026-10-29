# NuvioSwitch-0014 Account sync

## Why

Signing in (0013) is only useful if the Switch then shows the same add-ons, library, Continue Watching and watched marks as the user's other Nuvio devices. It should also report its own playback back to them. Without sync, an episode watched on the Switch would still look unwatched on NuvioTV.

## What Changes

- Sync four kinds of per-profile data with the Nuvio backend, using the same RPCs and payloads as NuvioTV's `core/sync/*` services:
  - **Add-ons:** push `sync_push_addons`; pull by reading the `addons` table filtered by `user_id` (the effective owner) and `profile_id`.
  - **Library items:** `sync_push_library_items`, `sync_delete_library_items`, the `sync_pull_library` snapshot, and the `sync_pull_library_delta` incremental pull with `sync_get_library_delta_cursor`.
  - **Watch progress:** `sync_push_watch_progress`, `sync_delete_watch_progress`, `sync_pull_watch_progress`, and `sync_pull_watch_progress_delta` with `sync_get_watch_progress_delta_cursor`.
  - **Watched items:** `sync_push_watched_items`, `sync_delete_watched_items`, `sync_pull_watched_items` (paged), and `sync_pull_watched_items_delta` with `sync_get_watched_items_delta_cursor`.
- Local edits are recorded in a per-profile pending-mutation queue that survives restarts and offline use. The queue is flushed when online, and pending local changes win over incoming remote ones until they are acknowledged.
- Every mutation sends a stable `p_origin_client_id` (`nuvio-switch-` plus 32 random characters), so the backend can tell which device made a change.
- When sync runs:
  - after sign-in and at launch: full pull, then flush;
  - on returning to the app: a delta pull, at least 2 minutes apart, after 2.5 s;
  - every 15 minutes while the app is open: a delta pull;
  - after playback stop and after library or watched changes: a push.
- Settings → Account shows when the last sync happened, with a Sync now action.

## Capabilities

### New Capabilities
- `nuvioswitch-account-sync`: what is synced, the RPC contract, the pending-mutation queue, how conflicts are resolved, and when sync runs.

### Modified Capabilities
None.

## Impact

- **Depends on:** 0003 (add-on store), 0009 (library, watch progress, watched items), 0011 (Settings), 0013 (Supabase client, session, effective user ID, profiles).
- **Code:**
  - new `src/core/sync/`: one service per data type, the mutation queue, cursors and a scheduler;
  - small hooks in the add-on, library and progress repositories so they record mutations;
  - per-profile stores `sync_state` (cursors, last pull time) and `sync_pending` (queued upserts and deletes).
- **Backend:** existing RPCs only.
- **Not synced yet:** plugins, collections, home catalog settings, the profile settings blob, and provider credentials. Their RPCs exist (`sync_push_plugins`, `sync_push_collections`, `sync_push_home_catalog_settings`, `sync_push_profile_settings_blob`, `sync_push_provider_credentials`), but the Switch has no matching features yet.
- **NuvioTV references:**
  - `core/sync/AddonSyncService.kt`, `core/sync/LibrarySyncService.kt`, `core/sync/library/*`, `data/remote/supabase/SupabaseLibrarySyncRemoteDataSource.kt`
  - `core/sync/WatchProgressSyncService.kt`, `core/sync/WatchedItemsSyncService.kt`, `core/sync/WatchStateMutationStore.kt`
  - `core/sync/SyncClientIdentity.kt`, `core/sync/StartupSyncService.kt`
