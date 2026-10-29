## 1. Sync core

- [ ] 1.1 Add `SyncClientIdentity` (global, `nuvio-switch-` + 32 chars, validated on load)
- [ ] 1.2 Add the per-profile `sync_state` and `sync_pending` stores and the pending-mutation API (record, list, acknowledge)
- [ ] 1.3 Add `SyncEngine` with the single-flight lock, the channel interface, and a full pull + flush routine
- [ ] 1.4 Add `sync_contract.hpp` with JSON builders and parsers for every RPC payload, plus unit tests pinning their shape

## 2. Channels

- [ ] 2.1 Add-ons channel: push `sync_push_addons`; pull from the `addons` table by effective owner and profile; respect "use primary add-ons"
- [ ] 2.2 Library channel: batched push and delete (500), paged snapshot, delta cursor and delta apply
- [ ] 2.3 Watch progress channel: push, delete by `progress_key`, snapshot, delta (900), newer `last_watched` wins
- [ ] 2.4 Watched items channel: push with null season and episode for movies, delete keys, paged snapshot (900), delta
- [ ] 2.5 Hook the add-on, library, progress and watched repositories so they record pending mutations
- [ ] 2.6 Unit tests for conflict rules and pending-wins behaviour with fake RPC responses

## 3. Scheduling and UI

- [ ] 3.1 Trigger a full pull and flush after sign-in, after a profile switch and at launch
- [ ] 3.2 Foreground delta pull (2.5 s delay, 2 min minimum gap) and a 15-minute periodic pull
- [ ] 3.3 Flush after playback stop and after library, watched or add-on changes
- [ ] 3.4 Settings → Account: last synced time, Sync now (30 s minimum gap), snapshot progress while it runs

## 4. Verification

- [ ] 4.1 Desktop: sign in to an account with data from NuvioTV and check that add-ons, library, Continue Watching and watched marks all appear
- [ ] 4.2 Desktop: change each of the four kinds on the Switch build and check it on NuvioTV after it syncs
- [ ] 4.3 Offline test: disconnect, watch and mark items, reconnect, and confirm the queue drains and nothing is duplicated
- [ ] 4.4 `ctest --test-dir build_desktop` passes (payload shapes, conflict rules, queue persistence)
- [ ] 4.5 Switch `.nro` builds; on hardware, leave via the HOME menu and return to confirm the foreground pull, and check that a large library snapshot doesn't freeze the UI
