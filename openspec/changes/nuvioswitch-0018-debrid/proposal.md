# NuvioSwitch-0018 Debrid

## Why

Many popular Stremio add-ons (Torrentio and similar) return mostly torrent streams. The Switch can't run a torrent engine, so from 0007 onward those streams show as "Needs debrid" and can't be played. Debrid services turn a cached torrent into a direct HTTPS link that mpv can play. NuvioTV resolves debrid on the device for Torbox and Premiumize, and for add-on `clientResolve` streams also Real-Debrid. Porting that makes most torrent add-ons usable on the Switch.

## What Changes

- **Connect a debrid service.**
  - **Torbox:** device code. `GET v1/api/user/auth/device/start?app=Nuvio`, then `POST v1/api/user/auth/device/token {device_code}`.
  - **Premiumize:** device code. `POST token` with `response_type=device_code` and `client_id=PREMIUMIZE_CLIENT_ID`, then `POST token` with `grant_type=device_code`, `code` and `client_id`.
  - Either can also take a pasted API key, checked with `GET v1/api/user/me` or `GET api/account/info`.
  - **Real-Debrid:** API key only, checked with `GET user`.
  - Keys are stored per profile, with a preferred resolver when more than one is set.
- **Cache check.** When the stream list loads, the info hashes of torrent streams are checked against the preferred Torbox or Premiumize account:
  - Torbox: `POST v1/api/torrents/checkcached?format=object {hashes}`;
  - Premiumize: `POST api/cache/check` with `items[]=magnet:?xt=urn:btih:<hash>`.
  - Streams get a Cached, Not cached or Unknown badge.
- **Resolve to a direct link** on selection, following `core/debrid/*DirectDebridResolver.kt`:
  - **Torbox:** `createtorrent` (`add_only_if_cached=true`, `allow_zip=false`), then `mylist?id&bypass_cache=true`, select the file, then `requestdl?token&torrent_id&file_id`.
  - **Premiumize:** `api/transfer/directdl` with `src=<magnet>`, then select the file and use its `link`.
  - **Real-Debrid** (only `clientResolve` streams with service `realdebrid`): `torrents/addMagnet`, `torrents/info/{id}`, `torrents/selectFiles/{id}`, `torrents/info/{id}`, `unrestrict/link`; the torrent is deleted on failure.
- **Magnet and file selection.**
  - Magnets are built with `DebridMagnetBuilder`: `xt=urn:btih:<hash>`, `dn=<filename>`, and `tr=` for each `tracker:` source (`dht:` sources skipped).
  - The file is chosen with `selectDebridFile`, in this order: exact filename, then the SxxEyy / NxM episode pattern, then `fileIdx`, then the largest playable video.
- Resolved links are cached per stream for the session, so retrying or picking the same stream doesn't re-resolve.
- **Settings.** A Debrid section with connect and disconnect per service, plus the preferred resolver.

## Capabilities

### New Capabilities
- `nuvioswitch-debrid`: debrid accounts, cache checks, magnet and file selection, and resolving to a direct URL.

### Modified Capabilities
- `nuvioswitch-stream-picker` (owned by NuvioSwitch-0007): the "Playability" requirement is `MODIFIED` so torrent-only streams show "Needs debrid" only when no debrid service is connected. The resolve flow itself is the `ADDED` requirement "Debrid-backed torrent playback". 0007 must be archived before this change so the modified block applies.
- `nuvioswitch-settings`: the Debrid section, as an `ADDED` delta.

## Impact

- **Depends on:** 0007 (stream picker, playability, player hand-off), 0008 (player), 0011 (Settings), 0013 (build-time `PREMIUMIZE_CLIENT_ID`; profiles).
- **Code:** new `src/core/debrid/`:
  - provider registry, per-profile `debrid_settings` store;
  - Torbox, Premiumize and Real-Debrid clients;
  - cache-check service, magnet builder, file selector, resolver with a session cache;
  - stream-picker badges and resolve step; a Settings section.
- **Config:** Premiumize device login is unavailable without `PREMIUMIZE_CLIENT_ID`; its API key entry still works. Torbox needs no client id.
- **Not included:** cloud library browsing, debrid stream formatter templates and sorting/filter preferences (`DebridStreamFormatter*`, `DebridStreamPresentation.kt`), instant-playback preparation, and a local torrent engine.
- **NuvioTV references:**
  - `core/debrid/DebridProvider.kt`, `DebridMagnetBuilder.kt`, `DebridFileSelection.kt`, `DirectDebridResolver.kt`
  - `core/debrid/TorboxDirectDebridResolver.kt`, `PremiumizeDirectDebridResolver.kt`, `RealDebridDirectDebridResolver.kt`, `LocalDebridService.kt`, `LocalDebridAvailabilityService.kt`
  - `data/remote/api/TorboxApi.kt`, `PremiumizeApi.kt`, `RealDebridApi.kt`
  - `ui/screens/settings/DebridSettingsViewModel.kt`, `data/local/DebridSettingsDataStore.kt`, `core/di/NetworkModule.kt` (base URLs)
