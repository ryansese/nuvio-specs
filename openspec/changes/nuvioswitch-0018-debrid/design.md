## Context

NuvioTV turns torrent streams into playable URLs in two ways (`core/debrid/DirectDebridResolver.kt`):
- **`clientResolve` streams.** Some add-ons return a `clientResolve` block naming a debrid service. NuvioTV resolves it on the device with the matching resolver (Torbox, Premiumize, Real-Debrid).
- **Local torrent resolve.** Plain `infoHash` streams are wrapped into a `clientResolve` for the user's preferred Torbox or Premiumize account (`resolveLocalTorrentStream`), after an optional cache check (`LocalDebridService.checkCached`).

All three resolvers use plain REST with bearer API keys. Torbox and Premiumize support device-code login (`DebridSettingsViewModel.startDeviceAuthorization` and `redeemDeviceAuthorization`). Real-Debrid is registered but hidden in NuvioTV's UI (`visibleInUi = false`), and it can resolve only `clientResolve` streams.

NuvioTV's local torrent engine (`core/torrent/`, a closed libtorrent AAR) is out of scope on the Switch, so debrid is the only way to play torrents.

## Goals / Non-Goals

**Goals:**
- Make torrent add-ons usable on the Switch through Torbox, Premiumize and Real-Debrid.
- Match NuvioTV's resolve, file-selection and magnet rules, so the same stream picks the same file.
- Cache badges, so users don't pick uncached torrents.

**Non-Goals:**
- Torbox and Premiumize cloud library browsing (`core/cloud/`).
- Debrid stream formatting templates, sort modes, quality, HDR, DV and codec filters, max results, and instant-playback preparation.
- Usenet and web downloads, and Real-Debrid cache checks (Real-Debrid has no instant-availability API in NuvioTV either).
- Syncing debrid keys through Nuvio `provider_credentials`.

## Decisions

**1. Provider registry mirrors `DebridProviders`.** It has ids, short names (TB, PM, RD), auth methods and capabilities (client resolve, local torrent cache check and resolve). Code checks capabilities, not provider ids, so adding a provider later is local.

**2. Pure, tested core.**
- `buildMagnet(stream)` mirrors `DebridMagnetBuilder.fromStream`.
- `selectDebridFile(files, hint)` copies `DebridFileSelection.kt` exactly: the order, uniqueness rule, regex `(?<![a-z0-9])(?:s0*Se0*E|0*Sx0*E)(?![0-9])` and extension list.
- `deviceTokenResult(provider, status, body)` maps device-token responses.

All three are unit-tested with NuvioTV-derived fixtures.

**3. Resolver pipeline.**
- `DebridResolver::resolve(stream, season, episode) -> Result{Success(url, filename, size) | NotCached | Stale | MissingKey | Error}`.
- It runs on the worker pool, and the stream picker shows a spinner while it works.
- Successes are cached for the session, keyed as `DirectDebridResolver.directDebridResolveCacheKey` does.

**4. Cache check runs after the list renders.** All torrent hashes from the loaded add-ons go in one batch call per refresh, and badges update in place. If the check fails, badges show Unknown and selection still tries to resolve, as NuvioTV does when `isCached` returns null.

**5. Real-Debrid is visible on the Switch.** Its API key entry appears in Settings, even though NuvioTV hides it. It resolves only `clientResolve` streams with service `realdebrid`; plain info hashes go to Torbox or Premiumize only. This follows the roadmap. See the open question.

**6. Multipart and form bodies.** Torbox `createtorrent` needs `multipart/form-data`, and Premiumize and Real-Debrid need `application/x-www-form-urlencoded`. The 0002 HTTP client gains small helpers for both through libcurl's mime API.

## Risks / Trade-offs

- [Provider APIs change, or rate-limit device polling] → Error mapping is in one place per provider, and polling follows the server's `interval`.
- [Large season packs: `mylist` responses can be big] → Parse only the file fields. Fine for Switch memory.
- [Uncached torrents: Torbox `add_only_if_cached=true` avoids queuing downloads; Premiumize `directdl` reports not cached] → Users get a clear message instead of a stall.
- [Real-Debrid leaves torrents in the account when resolution succeeds] → Same as NuvioTV; documented. Failures clean up.
- [API keys in plain JSON on the SD card] → Same trade-off as other tokens (0013). Masked in the UI and cleared on disconnect.

## Open Questions

- NuvioTV hides Real-Debrid in its UI. Should the Switch show it, as this change proposes, or keep it hidden for parity?
- Should the 0007 Playability requirement be formally amended (a `MODIFIED` delta) when 0007 is archived, or is the `ADDED` "Debrid-backed torrent playback" requirement enough? This change assumes an amendment at archive time.
