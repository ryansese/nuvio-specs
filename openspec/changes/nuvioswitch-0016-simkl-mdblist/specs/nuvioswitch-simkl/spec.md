## ADDED Requirements

### Requirement: Simkl availability and request metadata
Simkl SHALL be available only when the build has a non-empty `SIMKL_CLIENT_ID`. Every request to `https://api.simkl.com` SHALL include:
- query parameters `client_id=<SIMKL_CLIENT_ID>`, `app-name=<SIMKL_APP_NAME or "nuvio">` and `app-version=<app version>`;
- headers `User-Agent: NuvioSwitch/<version>` and `Accept: application/json`;
- `Authorization: Bearer <token>` when connected.

This follows `data/simkl/SimklApiMetadata.kt`.

#### Scenario: Not configured
- **WHEN** the build has no Simkl client id
- **THEN** Settings has no Simkl section and nothing is sent to Simkl

### Requirement: Simkl PIN connection
Connecting SHALL call `GET /oauth/pin` and require `result: "OK"`, `user_code`, `verification_url`, a positive `expires_in` and a positive `interval`. Otherwise it SHALL fail. The app SHALL show the code, a QR code of the URL and a countdown. It SHALL then poll `GET /oauth/pin/{user_code}` every `interval` seconds:
- `result: "OK"` with a non-blank `access_token` stores the token for the active profile, then fetches `POST /users/settings` for the display name;
- `result: "KO"` means pending;
- a local expiry ends the session as expired.

#### Scenario: Connect
- **WHEN** the user enters the PIN at simkl.com and approves
- **THEN** the next poll returns OK with a token, and Settings shows Simkl as connected

#### Scenario: PIN expires
- **WHEN** the countdown reaches zero before approval
- **THEN** polling stops and the screen offers a new PIN

### Requirement: Simkl scrobbling
When connected, the app SHALL send each scrobble event as `POST /scrobble/{start|pause|stop}` with body `{progress, movie | show | anime: {title, year, ids}, episode?: {season, number}}`, built as in `buildSimklScrobbleBody`. The ids include whichever of `imdb`, `tmdb`, `simkl`, `tvdb`, `mal`, `anidb`, `anilist` and `kitsu` the content id provides. A series event without an episode SHALL NOT be sent. A 409 on stop SHALL count as success. Scrobbles SHALL NOT be retried.

#### Scenario: Pause uses Simkl's pause endpoint
- **WHEN** the user pauses an IMDb movie at 50%
- **THEN** `POST /scrobble/pause` is sent with `progress: 50` and `movie.ids.imdb`

#### Scenario: Anime by kitsu id
- **WHEN** the user plays content with id `kitsu:1376` episode 3
- **THEN** the scrobble carries `anime.ids.kitsu = 1376` and the episode number 3
