## Context

After 0001, NuvioSwitch has a CMake project, an app shell with placeholder tabs, and a desktop-only doctest target. Nothing talks to the network or saves anything yet.

What NuvioTV does, in `core/di/NetworkModule.kt`:
- **Clients:** an OkHttp "validating" client for first-party APIs, and an `addonPermissive` client that accepts any certificate and hostname.
- **Common to both:**
  - 30 s connect and 60 s read timeouts;
  - `User-Agent: Nuvio/<ver>` and an `Accept-Language` from `buildAcceptLanguageHeader()`;
  - a 50 MB disk cache;
  - a network interceptor that rewrites 4xx/5xx responses to `Cache-Control: no-store`.
- **Errors:** `core/network/SafeApiCall.kt` and `NetworkResult.kt` turn exceptions and HTTP errors into `Success`/`Error(code, message)`.
- **Storage:** per-profile DataStore files are named `{feature}` for profile 1 and `{feature}_p{id}` for others (`data/local/ProfileDataStoreFactory.kt`). Values are mostly JSON strings.

Borealis constraints that shape this design:
- UI calls must happen on the main thread. `brls::sync(fn)` queues work to run before the next frame.
- `brls::async(fn)` runs on **one** background task-loop thread (`lib/core/thread.cpp`), so it can't run several HTTP requests in parallel.
- `brls::Image::setImageFromMem(data, size)` decodes and uploads a texture, so it must run on the UI thread.
- nlohmann/json is bundled under `include/borealis/extern/nlohmann`.

## Goals / Non-Goals

**Goals:**
- One HTTP API that every later feature uses (add-ons, catalogs, meta, streams, Supabase, Trakt), with NuvioTV's defaults.
- Durable, simple storage that later changes can build typed stores on (add-ons in 0003, library and progress in 0009).
- Poster loading that's good enough for scrolling rows of 126×189 images on the Switch's limited RAM.

**Non-Goals:**
- HTTP/2, ETag/`If-None-Match` revalidation and response streaming. Playback streaming is handled by mpv in 0008, not by this client.
- The IPv4-first DNS quirk (`core/network/IPv4FirstDns.kt`). Reconsider it only if IPv6 causes problems on the Switch.
- Typed stores for specific features; this change only provides the store primitive.
- Image resizing or format conversion. Images are shown as decoded by borealis.

## Decisions

**1. Own worker pool plus curl easy handles; results delivered with `brls::sync`.**
- `core::async::WorkerPool` has N `std::thread` workers (4 on Switch and desktop) and a FIFO queue.
- `core::net::HttpClient::send(Request, Callback) -> RequestHandle` queues a job that runs a curl easy handle synchronously on a worker, then posts the callback with `brls::sync`.
- `RequestHandle::cancel()` sets an atomic flag. Curl's progress callback checks it and aborts the transfer, and a cancelled job never posts its callback.
- Alternative: one `curl_multi` event loop thread. It's more efficient, but harder to get right and to test. Four blocking workers are plenty for an app making a handful of parallel add-on calls.
- Alternative: `brls::async`. Rejected, because it has a single thread.

**2. `curl_global_init` once at startup, one easy handle per request, and a shared connection cache.**
- A `CURLSH` share handle (with a mutex) shares DNS, connection and TLS session caches across workers, so repeated calls to the same add-on host reuse connections.
- Timeouts: `CURLOPT_CONNECTTIMEOUT=30`, plus `CURLOPT_LOW_SPEED_LIMIT=1` / `CURLOPT_LOW_SPEED_TIME=60` as the read-timeout equivalent.
- `CURLOPT_FOLLOWLOCATION`, `MAXREDIRS=10`, and `CURLOPT_ACCEPT_ENCODING=""` (gzip).
- Permissive mode sets `CURLOPT_SSL_VERIFYPEER=0` and `CURLOPT_SSL_VERIFYHOST=0`, as NuvioTV's add-on client does. It's chosen per request with `Request::permissiveTls`; add-on code in 0003 always sets it.

**3. Result type mirrors NuvioTV's `NetworkResult`.**
- `HttpResult { ok; status; headers; body; finalUrl; error }`.
- `error` is one of `None | Http | Timeout | Connect | Tls | Cancelled | Other`, plus a message.
- Non-2xx responses are errors with status and body, as `safeApiCall` treats them.

**4. Disk response cache: one file per URL plus a small JSON index.**
- The key is a hash of method + URL (FNV-1a 64 is enough; there's no security need). Files go in `cache/http/<hash>.body`, and `cache/http/index.json` records `{url, storedAt, maxAge, size, lastAccess}`.
- Freshness comes from `Cache-Control` `max-age` or `s-maxage`, by a pure parser that is unit-tested.
- Responses with `no-store`, `no-cache` or `private`, non-GET requests and non-2xx responses are never stored. That's simpler than rewriting headers the way NuvioTV's interceptor does, and gives the same behaviour.
- Total size is capped at 50 MB with LRU eviction, matching NuvioTV's `addon_http_cache`.
- Per request: `cachePolicy = Default | NoCache | StaleIfError`. `StaleIfError` returns an expired body flagged `stale=true` when the network fails. 0006 uses it to show details offline.
- Alternative: libcurl has no built-in HTTP cache, so this is the minimal option.

**5. JSON helpers over nlohmann.** Stremio JSON is loose: numbers come as strings, lists can be strings or objects, and fields may be null. `core::json` adds tolerant getters, unit-tested on the cases NuvioTV's mappers handle:
- `str(j, key)`
- `optStr`
- `strList` (accepts a string or an array, and drops non-strings)
- `optInt`, which accepts a number or a numeric string
- `optBool`, which accepts a bool, `"true"` or a non-zero number

**6. Storage: `core::storage::Store` is one JSON object per file.**
- `Store::open(name, Scope::Profile|Global)` loads the file lazily into memory. `get<T>(key)` and `set(key, value)` change it in memory, and `save()` queues an atomic write on one dedicated storage thread, so writes are serialised.
- The atomic write goes to `<file>.tmp`, then `fflush` and `fsync` where supported, then `rename`. On Switch's FAT/exFAT SD card, `rename` over an existing file can fail, so the fallback is to remove the old file and rename again. The tiny window that leaves is accepted (see Risks).
- `Paths::dataDir()`: on Switch `sdmc:/switch/NuvioSwitch/`; on macOS `$HOME/Library/Application Support/NuvioSwitch/`. `NUVIOSWITCH_DATA_DIR` overrides it, for tests and development.
- The active profile id comes from `ProfileContext::current()` and is fixed at `1` until 0013.

**7. Image loader: `core::image::ImageLoader::load(brls::Image*, url, placeholder)`.**
- Each target view gets a request token stored in a `std::unordered_map<brls::Image*, uint64_t>` that's touched only on the UI thread. A result is applied only if the token still matches.
- Views call `ImageLoader::cancel(view)` in `prepareForReuse()` and when they're destroyed.
- Downloads use their own `WorkerPool(4)`, separate from API calls, so posters never starve add-on requests.
- The memory LRU holds encoded bytes, capped at 32 MB; the disk cache in `cache/images/` is capped at 100 MB.
- Decoding happens in `setImageFromMem` on the UI thread. Borealis owns the textures, so the loader keeps bytes, not textures.
- Alternative: decode off-thread with stb_image and upload later. Rejected for now, because borealis has no public API for setting an RGBA buffer.

**8. Testability.** Code with no curl or borealis dependency lives in `src/core/` and is compiled into `NuvioSwitchTests`:
- the cache-control parser, cache keys and LRU index;
- the JSON helpers;
- `Paths`, using a temp directory;
- `Store` load/save and the corrupt-file handling;
- `Accept-Language` building.

The curl and borealis glue lives in `src/core/net/curl_*.cpp` and `src/core/image/*.cpp`, which are excluded from the test target.

## Risks / Trade-offs

- [`switch-curl` TLS: the portlib may need a CA bundle for verified requests] → Check at startup with a request to a known HTTPS endpoint. If needed, ship `cacert.pem` in romfs and set `CURLOPT_CAINFO` (NuvioTV's mpv path does the same with `tls-ca-file`).
- [Rename isn't atomic on FAT] → Use the remove-then-rename fallback and keep `.tmp` files. On startup, if `<store>.json` is missing but `<store>.json.tmp` exists and parses, recover from it.
- [RAM on Switch: about 3.2 GB for applications, less in applet mode] → Cap the memory caches (32 MB images) and load posters at medium size. Detect applet mode (`appletGetAppletType`) and halve the caches there.
- [Four blocking workers can be held by slow add-ons] → Timeouts bound this. Image downloads have their own pool.
- [Shared curl handles need lock callbacks] → Use one `std::mutex` per `curl_lock_data` type, as the libcurl share docs describe.

## Open Questions

- Whether the desktop build should use Homebrew curl or the macOS system curl. Default: whatever `find_package(CURL)` resolves; the system one is fine.
