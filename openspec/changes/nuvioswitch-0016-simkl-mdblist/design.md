## Context

0015 added `ScrobbleDispatcher` and the `Tracker` interface, with Trakt as the first tracker. NuvioTV's Simkl and MDBList support is large (`data/simkl/*`, `data/mdblist/*`: library projections, sync engines, reconciliation), but the scrobble and auth parts are small and self-contained:
- **Simkl** uses a PIN flow (`SimklAuthRepository.kt`). It signs every request with the `client_id`, `app-name` and `app-version` query parameters (`SimklApiMetadata.kt`) and has real start, pause and stop endpoints (`SimklMutationService.kt`).
- **MDBList** uses standard OAuth device authorisation with `scope=write` (`MdbListAuthRepository.kt`) and also has start, pause and stop endpoints (`MdbListScrobbleService.kt`).
- **MDBList ratings** (`MdbListRatingsClient.kt`, `MDBListApi.kt`) work either with a user API key passed as `apikey`, or with the OAuth account.

## Goals / Non-Goals

**Goals:**
- Scrobbling parity with NuvioTV for Simkl and MDBList.
- MDBList ratings on the details screen, the most visible MDBList feature in NuvioTV.

**Non-Goals:**
- Simkl and MDBList library sources, watchlists, lists, playback import and sync engines (`SimklSyncEngine.kt`, `MdbListSyncEngine.kt`).
- MDBList's `/scrobble/clear` on "remove from Continue Watching", which needs the MDBList playback snapshot that isn't ported.
- Simkl anime-id preference settings (`SimklAnimeIdPreference.kt`); default id selection is used.
- Syncing credentials through Nuvio `provider_credentials`.

## Decisions

**1. Two `Tracker` implementations, each with its own small client.**
- They share only the 0002 HTTP client and a `DeviceCodeView` UI component (code, QR, countdown) extracted from 0015's Trakt screen.
- Neither restarts on seek.

**2. Pure helpers for testing.**
- `SimklPinFlow`, `MdbListDeviceFlow` (error strings to states) and the two scrobble body builders are pure functions with unit tests.
- Expected bodies are copied from NuvioTV's builders: Simkl `progress` clamping and rounding (`clampAndRoundProgress`), and MDBList's 2-decimal round-down.

**3. Media id mapping.** One `TrackingIds` struct is parsed from the content id: `tt…` → imdb; `tmdb:`, `trakt:`, `simkl:`, `kitsu:`, `mal:` and `anilist:` prefixes; season and episode from the video id. Each tracker picks the ids it supports. The Simkl kind is movie, show, or anime (anime when the id is `kitsu:`, `mal:` or `anilist:` and there is no season).

**4. Ratings loader.** `MdbListRatingsLoader` prefers the account token when connected, otherwise the API key. It caches per item for the session, and is called from the details screen after the meta loads. It is off unless the toggle is on, matching NuvioTV's default (`mdblist_enabled = false`).

## Risks / Trade-offs

- [Simkl requires app registration, and the `app-name` must match it] → `SIMKL_APP_NAME` is configurable and defaults to `"nuvio"`, as in NuvioTV.
- [MDBList rate limits on free keys] → Ratings are fetched only when the details screen opens, then cached. 429 and `X-RateLimit-Remaining: 0` back off until `Retry-After`.
- [Anime numbering differs between services] → Use whatever ids the add-on gives. Mismatches are a known NuvioTV limitation too.

## Open Questions

- Should the MDBList API key be shared across profiles? NuvioTV stores it per profile; this change does the same.
