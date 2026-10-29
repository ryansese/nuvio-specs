## 1. Debrid core

- [ ] 1.1 Add the provider registry (ids, short names, auth methods, capabilities) and the per-profile `debrid_settings` store (keys, preferred resolver)
- [ ] 1.2 Add `buildMagnet` and `selectDebridFile`, ported exactly, with unit tests using NuvioTV-derived fixtures (filename, season pack, ambiguity, `fileIdx`, largest)
- [ ] 1.3 Add multipart and url-encoded form helpers to the HTTP client

## 2. Providers

- [ ] 2.1 Torbox client: `user/me`, device start and token, `checkcached`, `createtorrent`, `mylist`, `requestdl`
- [ ] 2.2 Premiumize client: device token (both steps), `account/info`, `cache/check`, `transfer/directdl`
- [ ] 2.3 Real-Debrid client: `user`, `addMagnet`, `info`, `selectFiles`, `unrestrict/link`, delete
- [ ] 2.4 Device-token result mapping for Torbox and Premiumize, with unit tests

## 3. Resolution

- [ ] 3.1 `DebridResolver`: clientResolve routing by service; local torrent resolve through the preferred Torbox or Premiumize account; result types; session cache
- [ ] 3.2 Cache availability check after the stream list renders; badges for Cached, Not cached and Unknown
- [ ] 3.3 Stream picker: make torrent streams selectable when debrid is connected, show the resolving spinner and failure messages, and hand off the resolved URL to the player

## 4. Settings

- [ ] 4.1 Debrid section: per-provider connect (device login or API key with validation), disconnect, masked key, preferred resolver

## 5. Verification

- [ ] 5.1 Desktop with Torbox: connect via device login; open a Torrentio movie and check Cached badges; play a cached stream; pick an uncached one and check the message
- [ ] 5.2 Desktop with Premiumize via API key: play an episode from a season pack and check the right file plays
- [ ] 5.3 Desktop with Real-Debrid: play a `clientResolve` stream for `realdebrid` and check it resolves
- [ ] 5.4 With no debrid connected, check torrent streams still show "Needs debrid"
- [ ] 5.5 `ctest --test-dir build_desktop` passes (magnet builder, file selection, token mapping)
- [ ] 5.6 Switch `.nro` builds; on hardware, resolve and play a 4K debrid stream and check playback starts and seeks
