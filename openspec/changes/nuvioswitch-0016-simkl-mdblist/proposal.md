# NuvioSwitch-0016 Simkl and MDBList

## Why

NuvioTV treats Trakt, Simkl and MDBList as equal trackers, and some Nuvio users track on Simkl (popular for anime) or MDBList instead of Trakt. MDBList also provides the multi-source ratings (IMDb, Rotten Tomatoes, Metacritic and others) that NuvioTV shows on its details screen. 0015 added a tracker-neutral scrobble dispatcher, so both services can plug into it.

## What Changes

- **Simkl.**
  - Connect with Simkl's PIN flow: `GET /oauth/pin` returns `user_code`, `verification_url`, `expires_in` and `interval`; then poll `GET /oauth/pin/{user_code}` until `result: "OK"` with an `access_token`. `KO` means pending, and expiry is tracked locally.
  - Every request carries the query parameters `client_id`, `app-name` and `app-version`, plus `Authorization: Bearer <token>`.
  - Scrobbling sends `POST /scrobble/{start|pause|stop}` with `{movie|show|anime, episode?, progress}`. Simkl has a real pause endpoint, and a 409 on stop counts as success.
- **MDBList.**
  - Connect with OAuth device authorisation: `POST /oauth/device-authorization/` with `scope=write`, then poll `POST /oauth/token/` with `grant_type=urn:ietf:params:oauth:grant-type:device_code`. Responses `authorization_pending`, `slow_down` (+5 s), `expired_token` and `access_denied` are handled. Tokens are refreshed with `grant_type=refresh_token`, and Disconnect revokes with `POST /oauth/revoke_token/`.
  - Scrobbling sends `POST /scrobble/{start|pause|stop}` with `{movie|show: {ids, season?, episode?}, progress}`. Progress is rounded down to 2 decimals.
- **MDBList ratings.**
  - An optional MDBList API key, entered with the on-screen keyboard, or the connected account, enables ratings on the details screen: `GET https://api.mdblist.com/{provider}/{type}/{id}?apikey=…`.
  - The ratings row shows IMDb, TMDB, Trakt, Letterboxd, Rotten Tomatoes (critics and audience), Metacritic and others, whichever MDBList returns.
- **Settings.** Simkl and MDBList sections with Connect, Connected and Disconnect, plus the MDBList API key field and a ratings toggle.
- Both trackers receive the same scrobble events as Trakt. Neither restarts on seek; only Trakt does, as in NuvioTV.

## Capabilities

### New Capabilities
- `nuvioswitch-simkl`: Simkl PIN connection and scrobbling.
- `nuvioswitch-mdblist`: MDBList device connection, scrobbling, and ratings on the details screen.

### Modified Capabilities
None. These are added as `ADDED` delta specs on capabilities owned by earlier changes:
- `nuvioswitch-playback`: delivering scrobbles to several trackers at once.
- `nuvioswitch-settings`: the Simkl and MDBList sections.

## Impact

- **Depends on:** 0015 (scrobble dispatcher and `Tracker` interface), 0006 (details screen for ratings), 0011 (Settings), 0013 (build-time `SIMKL_CLIENT_ID`, `SIMKL_APP_NAME`, `MDBLIST_CLIENT_ID`; profiles).
- **Code:** new `src/core/simkl/` and `src/core/mdblist/` (client, auth store, tracker, ratings loader), a ratings row on the details screen, and two Settings sections.
- **Config:** Simkl is disabled without `SIMKL_CLIENT_ID`. MDBList account features are disabled without `MDBLIST_CLIENT_ID`, but ratings still work with a user API key.
- **NuvioTV references:**
  - `data/simkl/SimklAuthRepository.kt`, `SimklApiMetadata.kt`, `SimklApiClient.kt`, `SimklMutationService.kt`, `SimklMutationBodies.kt`
  - `data/mdblist/MdbListAuthRepository.kt`, `MdbListScrobbleService.kt`, `MdbListMutationTarget.kt`, `MdbListRatingsClient.kt`, `MdbListHttpClient.kt`, `OkHttpMdbListEngine.kt`
  - `data/remote/api/MDBListApi.kt`, `data/repository/MDBListRepository.kt`, `data/local/MDBListSettingsDataStore.kt`
  - `core/tracking/TrackingWrites.kt`
