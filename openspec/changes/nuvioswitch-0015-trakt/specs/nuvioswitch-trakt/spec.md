## ADDED Requirements

### Requirement: Trakt availability
Trakt features SHALL be shown only when the build has non-empty `TRAKT_CLIENT_ID` and `TRAKT_CLIENT_SECRET`. All Trakt requests SHALL go to `https://api.trakt.tv/` with the headers `trakt-api-key: <TRAKT_CLIENT_ID>`, `trakt-api-version: 2` and `User-Agent: NuvioSwitch/<version>`.

#### Scenario: Not configured
- **WHEN** the build has no Trakt client id
- **THEN** Settings has no Trakt section, and details screens show no watchlist toggle

### Requirement: Device-code connection
Connecting SHALL `POST oauth/device/code {client_id}` and show `user_code` and `verification_url`, with a QR code of the URL and a countdown from `expires_in`. It SHALL then poll `POST oauth/device/token {code, client_id, client_secret}` every `interval` seconds. Poll responses SHALL be handled as in `data/repository/TraktAuthService.kt`:

| Response | Meaning | Action |
|---|---|---|
| 200 | approved | store `access_token`, `refresh_token`, `created_at`, `expires_in`, then fetch `GET users/settings` for the username |
| 400 | pending | keep polling |
| 404 | invalid code | stop and fail |
| 409 | already used | stop |
| 410 | expired | stop and show expiry |
| 418 | denied | stop and show denial |
| 429 | slow down | add 5 s to the interval, up to 60 s |

Tokens SHALL be stored per profile.

#### Scenario: User approves
- **WHEN** the user enters the code at trakt.tv/activate and approves
- **THEN** the next poll returns 200, and Settings shows "Connected as <username>"

#### Scenario: Slow down
- **WHEN** a poll returns 429 with the interval at 5 s
- **THEN** the next poll waits 10 s

#### Scenario: Profiles are separate
- **WHEN** profile 1 connects Trakt and the user switches to profile 2
- **THEN** profile 2 shows Trakt as not connected and doesn't scrobble

### Requirement: Token lifecycle
Before any authorised request, the app SHALL refresh the token if it expires within 60 seconds, using `POST oauth/token {refresh_token, client_id, client_secret, redirect_uri: TRAKT_REDIRECT_URI, grant_type: "refresh_token"}`, with refreshes serialised. A 401 SHALL force one refresh and one retry. If the refresh fails with 400, 401 or 403, the connection SHALL be cleared. A 429 on an authorised request SHALL be retried once after `Retry-After`, from 2 s up to a maximum of 60 s. Disconnect SHALL `POST oauth/revoke {token, client_id, client_secret}` and clear the tokens even if revoking fails.

#### Scenario: Expired access token
- **WHEN** the stored token expired yesterday and playback starts
- **THEN** the app refreshes the token first, then sends the scrobble with the new token

#### Scenario: Revoked on the website
- **WHEN** the user revoked the app on trakt.tv and the refresh returns 401
- **THEN** Trakt shows as disconnected in Settings, and scrobbling stops without errors during playback

### Requirement: Trakt identity of content
The app SHALL send to Trakt only content whose id is Trakt-compatible: `tt…` (IMDb), `tmdb:<n>` or `trakt:<n>`, as in `data/repository/TraktIdUtils.kt` `isTraktCompatibleId`. A movie is sent as `{movie: {title, year, ids}}`. An episode is sent as `{show: {title, year, ids}, episode: {season, number}}`, with the show ids taken from the series id and season and episode from the video id (`tt123:S:E`).

#### Scenario: IMDb episode
- **WHEN** the user plays video `tt0903747:2:5`
- **THEN** the Trakt body has `show.ids.imdb = "tt0903747"`, `episode.season = 2` and `episode.number = 5`

#### Scenario: Unsupported id
- **WHEN** the user plays content with id `kitsu:12345`
- **THEN** no Trakt request is made

### Requirement: Trakt scrobble delivery
On each scrobble event from the player, the app SHALL:
- send start as `POST scrobble/start`;
- send pause and stop as `POST scrobble/stop`;

with body `{movie|show+episode, progress, app_version}`, where progress is clamped to 0–100. It SHALL skip a send when the previous successful send had the same profile, item and action, was within 8 seconds and within 1.5 percentage points. It SHALL never skip a pause or stop that follows a start for the same item. A 2xx or 409 response is success. Stop SHALL be retried up to 2 more times on IO errors or 5xx, waiting 5 s after 502–504 and 1.5 s × attempt otherwise. Other 4xx SHALL NOT be retried.

#### Scenario: Pause records a paused playback
- **WHEN** the user pauses at 35%
- **THEN** `scrobble/stop` is sent with progress 35, and trakt.tv shows the item as paused

#### Scenario: Duplicate suppressed
- **WHEN** two pause events for the same episode at 40.0% and 40.8% are emitted 3 seconds apart
- **THEN** only the first is sent

#### Scenario: Server overloaded on stop
- **WHEN** `scrobble/stop` returns 503 twice and then 201
- **THEN** the app retried after 5 s each time, and the stop is recorded

### Requirement: History sync
When the user marks an item watched or unwatched on the details screen, and Trakt is connected, the app SHALL also `POST sync/history` (with `watched_at`) or `POST sync/history/remove`, sending a movie, a show with its seasons and episodes, or a single episode.

#### Scenario: Mark season watched
- **WHEN** the user marks season 1 of a show watched
- **THEN** one `sync/history` request lists every season 1 episode

### Requirement: Trakt watchlist
When Trakt is connected, the details screen SHALL show a Watchlist toggle that calls `POST sync/watchlist` or `POST sync/watchlist/remove`. The Library SHALL have a Trakt tab listing `GET sync/watchlist/movies` and `GET sync/watchlist/shows`, shown as posters that open the details screen for the item's IMDb id.

#### Scenario: Add to watchlist
- **WHEN** the user selects Add to Trakt watchlist on a movie's details screen
- **THEN** the movie appears in the Library's Trakt tab after it refreshes
