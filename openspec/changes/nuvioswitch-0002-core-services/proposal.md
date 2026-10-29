# NuvioSwitch-0002 Core services: HTTP, JSON, storage, async

## Why

Every feature after the app shell talks to the network and remembers something: add-on manifests, catalogs, posters, the library, watch progress. NuvioTV gets this from OkHttp, Moshi/Gson, DataStore and coroutines. NuvioSwitch needs the same foundations in C++ before 0003 (add-ons) can start, and they have to work the same way on macOS and the Switch.

## What Changes

- Add an HTTP client built on libcurl.
  - Requests run on a small worker pool, and results are delivered back on the UI thread.
  - It sends the same default headers as NuvioTV: `User-Agent` and `Accept-Language`.
  - Timeouts: 30 s to connect, 60 s to read.
  - A "permissive" mode that skips TLS checks, for add-on requests.
  - An on-disk response cache that honours `Cache-Control` and never stores 4xx/5xx responses.
- JSON handling uses the nlohmann/json copy already bundled with borealis, with small helpers for reading the loosely typed fields Stremio add-ons return.
- Add a key-value storage layer.
  - Each store is one JSON file, kept per profile, in the app's data folder (`sdmc:/switch/NuvioSwitch/` on Switch, the user's application-support folder on desktop).
  - Writes are atomic: write to a temp file, then rename.
- Add an image loader for posters and backdrops: fetched on the worker pool, kept in an in-memory LRU cache and a disk cache, applied to a borealis `Image` on the UI thread, with stale requests ignored when a view is reused.
- Expand the unit tests to cover the pure parts: cache-control parsing, cache keys, URL helpers, JSON helpers, storage paths and atomic writes.

## Capabilities

### New Capabilities
- `nuvioswitch-networking`: the HTTP client, the default headers, timeouts, TLS modes, the response cache, threading and how errors are reported.
- `nuvioswitch-storage`: the per-profile JSON key-value stores, their location on each platform, and atomic writes.
- `nuvioswitch-image-loading`: loading remote images into views with memory and disk caching.

### Modified Capabilities
None.

## Impact

- **Depends on:** NuvioSwitch-0001. It uses the CMake project, the `NuvioSwitchTests` target, the app shell and `version.hpp`.
- **Code:** new `src/core/net/`, `src/core/json/`, `src/core/storage/`, `src/core/image/` and `src/core/async/` in `repos/NuvioSwitch`. `main.cpp` gains startup and shutdown of the worker pool and storage.
- **Dependencies:**
  - libcurl: the Homebrew/system curl on macOS, the `switch-curl` portlib on Switch, both found through CMake.
  - nlohmann/json from `library/borealis/library/include/borealis/extern/nlohmann`.
  - libwebp for WebP posters (metahub serves only WebP): the `switch-libwebp` portlib on Switch, Homebrew `webp` on macOS.
  - No new UI libraries.
- **CI:** the Switch job installs `switch-curl`, which should already be in the devkitPro container, and the macOS job uses the system curl and installs `webp` with Homebrew.
- **NuvioTV references:**
  - `core/di/NetworkModule.kt` for the clients, headers, timeouts, the 50 MB caches, the `no-store` on errors and the permissive add-on client;
  - `core/network/SafeApiCall.kt` and `NetworkResult.kt` for how errors are wrapped;
  - `data/local/ProfileDataStoreFactory.kt` for per-profile stores.
