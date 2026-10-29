## 1. Shared pieces

- [ ] 1.1 Extract `DeviceCodeView` (code, QR, countdown, refresh) from the 0015 Trakt connect screen
- [ ] 1.2 Add `TrackingIds` parsing (imdb, tmdb, trakt, simkl, kitsu, mal, anilist, season and episode from the video id) with unit tests
- [ ] 1.3 Make the dispatcher deliver to every connected tracker independently, with seek restarts only for trackers that ask for them

## 2. Simkl

- [ ] 2.1 Add `SimklClient` with the required query parameters and headers
- [ ] 2.2 Implement `SimklPinFlow` (pin request validation, OK/KO polling, local expiry) and the connect screen
- [ ] 2.3 Fetch `/users/settings` after connecting for the display name, and store per profile
- [ ] 2.4 Simkl tracker: `POST /scrobble/{start|pause|stop}` body builder, 409-on-stop as success, no retries
- [ ] 2.5 Unit tests for the PIN flow and scrobble bodies (movie, show episode, anime)

## 3. MDBList

- [ ] 3.1 Add `MdbListClient` with headers, rate-limit back-off and form posts
- [ ] 3.2 Implement `MdbListDeviceFlow` (device authorisation, token polling, slow-down, expiry and denial), refresh, and revoke on disconnect
- [ ] 3.3 MDBList tracker: scrobble body with 2-decimal round-down; skip shows without an episode and unsupported ids
- [ ] 3.4 Ratings loader (API key or account, session cache) and the details-screen ratings row
- [ ] 3.5 Unit tests for the device flow mapping, scrobble bodies and ratings parsing

## 4. Settings

- [ ] 4.1 Simkl section (Connect, Connected, Disconnect)
- [ ] 4.2 MDBList section (Connect, Disconnect, API key with `/user` check, ratings toggle off by default)

## 5. Verification

- [ ] 5.1 Desktop: connect Simkl, pause and stop a movie, and check simkl.com shows the playback and watched state
- [ ] 5.2 Desktop: connect MDBList, scrobble an episode, and check mdblist.com; enter an API key and check ratings appear on details
- [ ] 5.3 With Trakt and Simkl both connected, block Simkl's host and check Trakt still gets its scrobbles
- [ ] 5.4 `ctest --test-dir build_desktop` passes (id parsing, auth flows, scrobble bodies, ratings parsing)
- [ ] 5.5 Switch `.nro` builds; on hardware, connect both services by phone and confirm scrobbles during playback
