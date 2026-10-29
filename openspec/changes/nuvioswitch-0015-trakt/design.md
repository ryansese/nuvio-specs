## Context

NuvioTV's Trakt support spans several services. This change ports a console-sized subset:
- auth: `TraktAuthService.kt`, with device code, refresh 60 s early, a circuit breaker and 429 handling;
- scrobbling: `TraktScrobbleService.kt`, with de-duplication, retries and 409 counting as success;
- watchlist: `TraktLibraryService.kt`;
- history: `TraktProgressService.kt`.

The player drives every tracker through `core/tracking/TrackingScrobbleCoordinator.kt`. It sends one event (start, pause or stop with a progress) to each connected tracker. Each tracker declares whether it wants the stop-and-restart on seek; only Trakt does (`TraktTrackingScrobbler.kt`: `STOP_AND_RESTART`).

**The pause question from the roadmap is settled.** `TraktApi.kt` only defines `scrobble/start` and `scrobble/stop`. `TraktScrobbleService.sendScrobble` routes `"start"` to `scrobbleStart` and everything else, including `"pause"`, to `scrobbleStop` (lines 101–104). Trakt treats a stop under 80% as a paused playback, so the result is the same as calling Trakt's own pause endpoint.

## Goals / Non-Goals

**Goals:**
- Reliable scrobbling with exactly NuvioTV's start, pause and stop semantics, so Trakt history matches whatever device was used.
- A tracker-neutral dispatch point in the player that 0016 reuses for Simkl and MDBList.
- Basic Trakt library value: watchlist, plus history kept in sync with mark watched.

**Non-Goals:**
- Importing Trakt's playback progress and "up next" into Continue Watching (`TraktProgressService` merge). The local and Nuvio-synced progress stays the source.
- Trakt lists, comments, related titles, recommendations and stats.
- Syncing the Trakt token through Nuvio `provider_credentials` (see Open Questions).
- Episode remapping between numbering schemes (`TraktEpisodeMappingService.kt`); add-on season and episode numbers are used as they are.

## Decisions

**1. `ScrobbleDispatcher` in `src/core/tracking/`.**
- The player calls it with `ScrobbleEvent{action, progress, item}`.
- The dispatcher holds the per-item state (start requested, completion sent) and applies the 80% and 99.5% rules.
- It then calls each registered `Tracker` that `isConnected()`, on a worker.
- Each tracker exposes `restartsOnSeek()`.
- Alternative: let each tracker watch the player. Rejected; it would duplicate the thresholds in every tracker.

**2. `TraktClient` on the 0002 HTTP client**, with a shared `executeAuthorized(request)` that does:
- proactive refresh;
- 401 → force refresh → retry once;
- 429 → wait for `Retry-After` (2–60 s) → retry once;
- a 5-minute circuit breaker after repeated auth failures, mirroring `TraktAuthService.kt` lines 50–110.

Refresh is guarded by a mutex.

**3. Pure state machines for testing.**
- `TraktDeviceAuthFlow` maps poll status codes to states.
- `ScrobbleDedup` implements the 8 s / 1.5% rule.
- `ScrobbleRules` applies the player thresholds.

All three are unit-tested on desktop.

**4. Per-profile `trakt` store** holding `access_token`, `refresh_token`, `created_at`, `expires_in`, `username` and the device-flow session. A pending device flow survives leaving the screen, as NuvioTV reuses it until expiry (`startDeviceAuth` lines 130–155).

**5. Library Trakt tab.** It is fetched when opened and on pull-to-refresh, and cached for 5 minutes. Items map to Stremio ids through `normalizeContentId`: IMDb first, then `tmdb:`, then `trakt:`. Details open with the default meta add-on resolution from 0006.

## Risks / Trade-offs

- [A Trakt app approval is needed for a new client] → Use the same `TRAKT_CLIENT_ID` as NuvioTV if its owner agrees; otherwise register a NuvioSwitch app. Either way it comes from config, and is never committed.
- [Rate limits: Trakt allows 1,000 GETs per 5 min] → Watchlist fetches are cached. Scrobbles are writes and rare.
- [Season/episode numbering differs between add-ons and Trakt (anime)] → Non-Trakt ids are skipped; accept rare mismatches for IMDb ids, as remapping is a non-goal.

## Open Questions

- Should the Trakt token be pulled from Nuvio `sync_pull_provider_credentials` when the account is signed in, so connecting once on NuvioTV also connects the Switch? That needs the credential payload format, and a decision on whether the Switch should hold another device's token.
