# NuvioSwitch-0015 Trakt

## Why

Many Nuvio users track what they watch on Trakt, and NuvioTV scrobbles to it automatically. Without Trakt, anything watched on the Switch is missing from their Trakt history. Trakt's device-code login suits a console: the user types a short code on their phone.

## What Changes

- **Connect Trakt.** Sign in with Trakt's device-code OAuth (`POST oauth/device/code`, then poll `POST oauth/device/token`). Tokens are stored per profile and refreshed through `POST oauth/token` 60 s before they expire. Disconnect revokes with `POST oauth/revoke`. Every request sends the headers `trakt-api-key: TRAKT_CLIENT_ID` and `trakt-api-version: 2`.
- **Scrobble playback.** A tracker-neutral scrobble dispatcher is added to the player. It emits start, pause and stop with a progress percentage, using NuvioTV's rules from `ui/screens/player/PlayerRuntimeControllerPlaybackEvents.kt`:
  - no start at 80% or more;
  - completion stop is sent once;
  - natural end sends stop at 99.5%;
  - a seek below 80% sends stop and then start again.
- **Trakt scrobble behaviour.**
  - Trakt receives start via `scrobble/start`. Pause and stop both go to `scrobble/stop`. This resolves the roadmap's open item: NuvioTV's `TraktScrobbleService.kt` has no pause endpoint; `scrobblePause` posts to `scrobble/stop` with the current progress, and Trakt saves it as a paused playback because progress is under 80%.
  - Duplicate sends are suppressed with the same window as NuvioTV: same item, same action, within 8 s and within 1.5% progress.
  - A 409 response counts as success.
  - Stop is retried on IO errors and 5xx.
- **History and watchlist.**
  - Mark watched and unwatched from the details screen also go to Trakt (`POST sync/history`, `POST sync/history/remove`).
  - A Watchlist toggle on the details screen uses `POST sync/watchlist` and `sync/watchlist/remove`.
  - The Library gets a Trakt tab listing the watchlist (`GET sync/watchlist/{movies|shows}`).
- **Settings.** A Trakt section shows Connect, the signed-in username (`GET users/settings`) and Disconnect.

## Capabilities

### New Capabilities
- `nuvioswitch-trakt`: Trakt device auth, the token lifecycle, scrobble delivery, history, the watchlist and the Library Trakt tab.

### Modified Capabilities
None. These are added as `ADDED` delta specs on capabilities owned by earlier changes:
- `nuvioswitch-playback`: the tracker-neutral scrobble events emitted by the player.
- `nuvioswitch-settings`: the Trakt section.

## Impact

- **Depends on:** 0008 (player), 0009 (watch progress and watched marks), 0006 (details screen), 0011 (Settings), 0013 (build-time `TRAKT_CLIENT_ID`/`TRAKT_CLIENT_SECRET`/`TRAKT_REDIRECT_URI`, profiles).
- **Code:** new `src/core/tracking/` (the scrobble dispatcher and tracker interface) and `src/core/trakt/` (API, auth store, scrobbler, library service); player hooks; details actions; a Library tab; a Settings section.
- **Config:** Trakt is disabled when `TRAKT_CLIENT_ID` or `TRAKT_CLIENT_SECRET` is empty.
- **NuvioTV references:**
  - `data/remote/api/TraktApi.kt`, `data/remote/dto/trakt/*`
  - `data/repository/TraktAuthService.kt`, `TraktScrobbleService.kt`, `TraktIdUtils.kt`, `TraktLibraryService.kt`, `TraktProgressService.kt`
  - `core/tracking/TrackingScrobbleCoordinator.kt`, `core/tracking/TrackingWrites.kt`
  - `ui/screens/player/PlayerRuntimeControllerPlaybackEvents.kt` (lines 830–995)
  - `core/di/NetworkModule.kt` (headers)
