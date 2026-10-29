## Context

NuvioTV syncs each data type with its own service over Supabase RPCs (`core/sync/*`). The model is:
- push local changes from a pending-mutation store;
- pull a full snapshot once, then switch to an event-log delta pull keyed by `event_id` (`sync_pull_*_delta`, with `sync_get_*_delta_cursor` to get the starting point);
- tag every write with `p_origin_client_id`, so a device can recognise its own changes.

`StartupSyncService.kt` controls when this runs: startup pull, foreground pulls (2.5 s delay, 2 min minimum gap), periodic pulls (15 min) and forced resyncs (30 s minimum).

On the Switch, 0003 and 0009 already store add-ons, library, progress and watched items in per-profile JSON stores. 0013 provides the authenticated Supabase client, the effective owner ID and profile switching.

## Goals / Non-Goals

**Goals:**
- Interoperate exactly with NuvioTV, NuvioMobile and NuvioDesktop for the four data kinds.
- Handle offline use and crashes without losing or duplicating changes.
- Keep it cheap on the Switch: incremental pulls, small batches, no realtime socket.

**Non-Goals:**
- Supabase Realtime subscriptions (`requestRealtimeSurfacePull`). Polling every 15 minutes is enough for a console.
- Syncing plugins, collections, home catalog settings, the profile settings blob or provider credentials.
- Merging Trakt, Simkl or MDBList library sources. That is 0015/0016, and only local library items sync here.

## Decisions

**1. Same wire format as NuvioTV, byte for byte where it matters.**
- Field names, units (milliseconds), key formats (`progress_key`) and page sizes (library 500; progress and watched 900) are copied from the services listed in the proposal.
- Reason: the other clients read the same rows, so any drift shows up there as broken data.

**2. One `SyncEngine` with a pluggable `SyncChannel` per data type.**
- Each channel implements `pushPending()`, `pullSnapshot()`, `pullDelta(cursor)` and `applyRemote()`.
- The engine owns the per-profile store `sync_state` (`{channel: {cursor, last_pull_ms}}`), the single-flight lock and the scheduler.
- Alternative: four independent services, as NuvioTV has. Rejected; on one thread pool, a shared engine keeps a single lock and log, with less code.

**3. The pending queue is stored per profile.**
- `sync_pending` holds `{channel: {upserts: {key: payload}, deletes: [key]}}`.
- Repositories write to it in the same store transaction as the local change: 0002's atomic write applies to each file, and the queue is written first.
- Acknowledged entries are removed after the RPC returns 2xx.

**4. Conflict rule.**
- Progress: the newer `last_watched` wins, and pending local keys are never overwritten.
- Library and watched items: a delete or upsert pending locally wins until it is acknowledged; otherwise remote events apply in `event_id` order.
- This matches NuvioTV's reconciliation (`WatchedItemsSyncService.kt` 330–390, `LibrarySyncReducer`).

**5. Add-ons use a table read, not a delta.** NuvioTV pulls them with `postgrest.from("addons").select { user_id, profile_id }`. The list is small, so a full replace on each pull is fine.

**6. Foreground detection.** Use borealis's application focus events, and on Switch the applet focus state from libnx (`appletGetFocusState` through borealis platform hooks), to trigger the foreground pull.

## Risks / Trade-offs

- [Schema drift: the backend changes an RPC signature] → Keep the payloads in one `sync_contract.hpp` with unit tests that pin the JSON shape, and check against NuvioTV when bumping the submodule.
- [A large first snapshot on slow Wi-Fi] → Page it, show progress in Settings, and keep the UI usable throughout.
- [Clock skew affecting the `last_watched` comparison] → Accepted, as NuvioTV has the same behaviour. The console clock is NTP-synced when online.
- [Two Switch installs with the same SD card copy would share one origin client id] → Edge case; documented.

## Open Questions

- Should Trakt and Simkl provider credentials (`sync_pull_provider_credentials`) be pulled once 0015/0016 land, so connecting on the TV also connects the Switch? Deferred to those changes.
