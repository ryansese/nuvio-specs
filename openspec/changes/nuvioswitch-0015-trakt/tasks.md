## 1. Scrobble dispatch (tracker-neutral)

- [ ] 1.1 Add the `Tracker` interface and `ScrobbleDispatcher` in `src/core/tracking/`
- [ ] 1.2 Implement `ScrobbleRules`: start under 80%, pause and stop gating, completion once, 99.5% on natural end, stop-and-restart on seek between 1% and 80%, short streams skipped
- [ ] 1.3 Hook the player: play/resume, pause, exit or stream switch, natural end, seek completed
- [ ] 1.4 Unit tests for `ScrobbleRules` covering every scenario in the playback spec

## 2. Trakt client and auth

- [ ] 2.1 Add `TraktClient` with base headers, `executeAuthorized`, proactive refresh, 401 retry, 429 `Retry-After` and a circuit breaker
- [ ] 2.2 Add the per-profile `trakt` store for tokens, username and device-flow session
- [ ] 2.3 Implement `TraktDeviceAuthFlow` (code request, polling with the status-code table, slow-down interval) and the connect screen with code, QR and countdown
- [ ] 2.4 Disconnect with `oauth/revoke`, then clear local tokens
- [ ] 2.5 Unit tests for the poll status mapping and refresh timing

## 3. Trakt features

- [ ] 3.1 Trakt id mapping (`isTraktCompatibleId`, show and episode from the video id) and body builders, with unit tests
- [ ] 3.2 Trakt tracker: start to `scrobble/start`; pause and stop to `scrobble/stop`; de-duplication; 409 as success; stop retries
- [ ] 3.3 History: send mark watched and unwatched to `sync/history` and `sync/history/remove`
- [ ] 3.4 Watchlist toggle on the details screen and the Library Trakt tab (`sync/watchlist/{movies,shows}`, 5-minute cache)
- [ ] 3.5 Settings → Trakt section (Connect, Connected as, Disconnect)

## 4. Verification

- [ ] 4.1 Desktop with a Trakt test app: connect, play a movie to 30% and pause, check trakt.tv shows it paused; finish and check it appears in history
- [ ] 4.2 Desktop: play an IMDb-id episode, check the correct season and episode are scrobbled; play a `kitsu:` id and check no request is made
- [ ] 4.3 Revoke the app on trakt.tv, then play something and check Settings shows disconnected with no player errors
- [ ] 4.4 `ctest --test-dir build_desktop` passes (scrobble rules, de-duplication, device-auth mapping, id mapping)
- [ ] 4.5 Switch `.nro` builds; on hardware, connect via code on a phone and confirm scrobbles during real playback
