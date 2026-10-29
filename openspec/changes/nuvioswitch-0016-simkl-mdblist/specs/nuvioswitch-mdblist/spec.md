## ADDED Requirements

### Requirement: MDBList device connection
When the build has a non-empty `MDBLIST_CLIENT_ID`, the app SHALL connect an MDBList account with OAuth device authorisation against `https://api.mdblist.com`:
- `POST /oauth/device-authorization/` (form: `client_id`, `scope=write`), returning `device_code`, `user_code`, `verification_uri`, the optional `verification_uri_complete`, `expires_in` and `interval` (1–3600);
- poll `POST /oauth/token/` (form: `client_id`, `grant_type=urn:ietf:params:oauth:grant-type:device_code`, `device_code`, `scope=write`).

Poll errors SHALL be handled as in `data/mdblist/MdbListAuthRepository.kt`:
- `authorization_pending`: keep polling;
- `slow_down`: add 5 s to the interval, up to 3600;
- `expired_token`: expired;
- `access_denied`: denied.

Tokens are stored per profile and refreshed with `grant_type=refresh_token`. Disconnect SHALL `POST /oauth/revoke_token/` (form: `client_id`, `token=<refresh token>`, `token_type_hint=refresh_token`) and clear local tokens either way. Requests send `Accept: application/json`, `User-Agent: NuvioSwitch/<version>` and `Authorization: Bearer <token>`. A 429, or `X-RateLimit-Remaining: 0`, SHALL defer further requests until `Retry-After`.

#### Scenario: Connect MDBList
- **WHEN** the user opens `verification_uri_complete` from the QR code and approves
- **THEN** the token poll succeeds, and Settings shows MDBList as connected

#### Scenario: Slow down
- **WHEN** a poll returns `slow_down` with a 5 s interval
- **THEN** the next poll waits 10 s

### Requirement: MDBList scrobbling
When connected, the app SHALL send each scrobble event as `POST /scrobble/{start|pause|stop}` with body `{movie | show: {ids, season?, episode?}, progress}`, where progress is clamped to 0–100 and rounded down to 2 decimals (`MdbListMutationTarget.scrobbleBody`). Events SHALL be sent only for movies and episodes with known season and episode numbers, and only for content with an id MDBList accepts (IMDb, TMDB, Trakt or TVDB).

#### Scenario: Episode scrobble
- **WHEN** the user stops `tt0903747:2:5` at 42.567%
- **THEN** `POST /scrobble/stop` is sent with `show.ids.imdb = "tt0903747"`, season 2, episode 5 and `progress: 42.56`

#### Scenario: Whole show not scrobbled
- **WHEN** the scrobble event has no episode number for a series
- **THEN** nothing is sent to MDBList

### Requirement: MDBList ratings on details
When MDBList ratings are turned on in Settings and either an API key is set or an MDBList account is connected, the details screen SHALL fetch ratings:
- with an API key: `GET /{provider}/{type}/{id}?apikey=<key>&append_to_response=keyword`;
- with an account: the same path, authenticated with the token;

where provider is `imdb` or `tmdb` and type is `movie` or `show`. The returned ratings SHALL be shown as a row of source and score items in place of the add-on's IMDb rating. Results SHALL be cached in memory per item for the session. Failures SHALL leave the add-on rating visible.

#### Scenario: Ratings shown
- **WHEN** ratings are on with a valid API key and the user opens a movie with id `tt0111161`
- **THEN** the details screen shows the IMDb, Rotten Tomatoes, Metacritic and other scores MDBList returns

#### Scenario: Invalid key
- **WHEN** the API key is rejected
- **THEN** the details screen shows the add-on's rating and Settings marks the key as invalid
