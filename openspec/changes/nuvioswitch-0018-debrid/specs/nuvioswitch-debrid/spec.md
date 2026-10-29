## ADDED Requirements

### Requirement: Debrid providers and credentials
The app SHALL support these providers. All requests use `Authorization: Bearer <key>`.

| Provider | Base URL | Sign-in methods |
|---|---|---|
| Torbox (`torbox`) | `https://api.torbox.app/` | device code or API key |
| Premiumize (`premiumize`) | `https://www.premiumize.me/` | device code or API key |
| Real-Debrid (`realdebrid`) | `https://api.real-debrid.com/rest/1.0/` | API key |

Keys SHALL be stored per profile in a `debrid_settings` store, along with a preferred resolver id. When the preferred provider has no key, the first configured provider that can resolve local torrents (Torbox, then Premiumize) is used.

#### Scenario: Two services configured
- **WHEN** Torbox and Premiumize both have keys and the preferred resolver is Premiumize
- **THEN** cache checks and torrent resolution use Premiumize

### Requirement: Device-code sign-in
For Torbox, the app SHALL call `GET v1/api/user/auth/device/start?app=Nuvio` and show `code` and `verification_url` (with the friendly URL when present). It SHALL then poll `POST v1/api/user/auth/device/token {device_code}` every `interval` seconds (at least 1, default 5):
- success with an access token stores it as the key;
- messages containing "pending" or "scan the code" mean pending;
- "expired" or HTTP 410 means expired;
- HTTP 404, 409 or 425 mean pending.

For Premiumize, the app SHALL `POST token` (form `response_type=device_code`, `client_id`) and show `user_code` and `verification_uri`. It SHALL then poll `POST token` (form `grant_type=device_code`, `code`, `client_id`):
- `access_token` stores it as the key;
- `authorization_pending` and `slow_down` mean pending;
- `invalid_grant` and `expired_token` mean expired;
- `access_denied` means failed.

#### Scenario: Torbox device login
- **WHEN** the user approves the code shown on the Switch at Torbox's verification URL
- **THEN** a later poll returns the access token, and Settings shows Torbox as connected

#### Scenario: Premiumize without client id
- **WHEN** the build has no `PREMIUMIZE_CLIENT_ID`
- **THEN** Premiumize offers API key entry only

### Requirement: API key entry and validation
The app SHALL accept a pasted or typed API key per provider and check it before saving:
- Torbox: `GET v1/api/user/me`
- Premiumize: `GET api/account/info`
- Real-Debrid: `GET user`

An invalid key SHALL NOT be saved.

#### Scenario: Wrong key
- **WHEN** the user enters a Real-Debrid key that `GET user` rejects with 401
- **THEN** the key is not saved and an error is shown

### Requirement: Magnet construction
When a stream has no magnet URI, the app SHALL build one as `magnet:?xt=urn:btih:<infoHash>`, followed by `&dn=<url-encoded behaviorHints.filename>` when there is a filename, and `&tr=<url-encoded tracker>` for each distinct `sources` entry. A `tracker:` prefix is stripped, and `dht:` entries and blanks are skipped.

#### Scenario: Torrentio-style stream
- **WHEN** a stream has `infoHash` "abc", `behaviorHints.filename` "Show S01E02.mkv" and sources `["tracker:udp://t.example:80", "dht:abc"]`
- **THEN** the magnet is `magnet:?xt=urn:btih:abc&dn=Show+S01E02.mkv&tr=udp%3A%2F%2Ft.example%3A80`

### Requirement: File selection
From the provider's file list, the app SHALL consider only playable video files (`.mp4`, `.mkv`, `.webm`, `.avi`, `.mov`, `.m4v`, `.ts`, `.m2ts`, `.wmv`, `.flv`) and pick one as `selectDebridFile` does:
1. A unique match on the stream's filename, by full path then by basename, case-sensitive then case-insensitive.
2. Otherwise a unique match on the episode pattern `S<season>E<episode>` or `<season>x<episode>` (leading zeros allowed, case-insensitive) in the basename.
3. If a filename or episode was known but nothing matched uniquely, no file.
4. Otherwise the file at `fileIdx` if it is playable.
5. Otherwise the largest playable file.

#### Scenario: Season pack
- **WHEN** a season pack holds `Show.S01E01.mkv` to `Show.S01E10.mkv` and the user plays season 1 episode 4 without a filename hint
- **THEN** `Show.S01E04.mkv` is selected

#### Scenario: Ambiguous match
- **WHEN** two files both match `S01E04`
- **THEN** resolution fails as stale instead of guessing

### Requirement: Cache availability check
When the stream list contains torrent streams and Torbox or Premiumize is the active resolver, the app SHALL check their lower-cased info hashes in one batch:
- Torbox: `POST v1/api/torrents/checkcached?format=object {hashes}`
- Premiumize: `POST api/cache/check` with `items[]=magnet:?xt=urn:btih:<hash>`

Each stream SHALL be marked Cached (with the cached name and size when given), Not cached, or Unknown when the check failed. The check SHALL NOT delay showing the stream list.

#### Scenario: Badges appear
- **WHEN** a stream list with 20 torrent streams loads and Torbox reports 6 hashes cached
- **THEN** those 6 streams show a Cached badge and the others show Not cached

### Requirement: Resolve to a direct URL
When the user selects a torrent stream with a debrid service available, the app SHALL resolve it to a direct HTTPS URL:
- **Torbox:** `POST v1/api/torrents/createtorrent` (multipart `magnet`, `add_only_if_cached=true`, `allow_zip=false`), then `GET v1/api/torrents/mylist?id=<id>&bypass_cache=true`, select the file, then `GET v1/api/torrents/requestdl?token=<key>&torrent_id=<id>&file_id=<file>&zip_link=false&redirect=false`, and use `data` as the URL.
- **Premiumize:** `POST api/transfer/directdl` (form `src=<magnet>`), select the file from `content`, and use its `link`. A `status: "error"` whose message mentions cache or not found means Not cached.
- **Real-Debrid:** only for streams whose `clientResolve.service` is `realdebrid`. `POST torrents/addMagnet`, `GET torrents/info/{id}`, select the file, `POST torrents/selectFiles/{id}` with `files=<file id>` (200 or 202), `GET torrents/info/{id}`, and require `status: "downloaded"`. Then `POST unrestrict/link` with the first link, and use `download`. On failure the torrent SHALL be deleted with `DELETE torrents/delete/{id}`.

Streams already marked Not cached SHALL fail fast with "Not cached on <service>". Successful results SHALL be cached for the session, keyed by service, info hash or magnet, file index, season and episode. The resolved stream keeps its filename and size, and SHALL be handed to the player like any HTTP stream.

#### Scenario: Play a cached torrent
- **WHEN** the user selects a Cached torrent stream with Torbox connected
- **THEN** the app resolves a Torbox download URL and the player starts it

#### Scenario: Not cached
- **WHEN** the user selects a stream marked Not cached
- **THEN** no resolve request is made and a "Not cached on Torbox" message is shown

#### Scenario: Retry uses cache
- **WHEN** a resolved stream fails to play and the user picks it again in the same session
- **THEN** the cached URL is reused without calling the debrid API
