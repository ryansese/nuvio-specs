## 1. Build wiring

- [ ] 1.1 Find libcurl in CMake for desktop (`find_package(CURL)`) and Switch (`switch-curl` portlib with its mbedtls/zlib dependencies); link it into the app only
- [ ] 1.2 Add the nlohmann include path from borealis to the app and to `NuvioSwitchTests`
- [ ] 1.3 Install `switch-curl` in the CI Switch job if the container lacks it

## 2. Async and JSON

- [ ] 2.1 Add `core::async::WorkerPool` (N threads, FIFO queue, clean shutdown) and a `postToUi` wrapper over `brls::sync`
- [ ] 2.2 Add `core::json` tolerant getters (`str`, `optStr`, `strList`, `optInt`, `optBool`) with unit tests that mirror NuvioTV mapper cases

## 3. Networking

- [ ] 3.1 Add `HttpResult`, `Request` (method, url, headers, body, `permissiveTls`, `cachePolicy`, timeouts) and `RequestHandle` with `cancel()`
- [ ] 3.2 Add the `Accept-Language` builder and default headers; unit-test `it-IT`, `en-US` and empty locale
- [ ] 3.3 Implement the curl executor: share handle with locks, timeouts, redirects, gzip, permissive TLS, cancellation via the progress callback, error mapping
- [ ] 3.4 Add the `Cache-Control` parser and HTTP disk cache (index, LRU eviction at 50 MB, no-store rules, `StaleIfError`); unit-test the parser and eviction
- [ ] 3.5 Add `HttpClient` that ties together the pool, cache and executor; call `curl_global_init`/`cleanup` from `main.cpp`
- [ ] 3.6 Verify TLS on Switch against a known HTTPS endpoint; ship `cacert.pem` in romfs and set `CURLOPT_CAINFO` if needed

## 4. Storage

- [ ] 4.1 Add `core::storage::Paths` (Switch, macOS, `NUVIOSWITCH_DATA_DIR` override) and create the folders on first use
- [ ] 4.2 Add `Store` (global or profile scope, get/set, `save()` on a dedicated writer thread, atomic write with FAT fallback, `.tmp` recovery, `.corrupt` handling)
- [ ] 4.3 Add `ProfileContext` with a fixed profile id `1`
- [ ] 4.4 Unit-test round trip, profile isolation, corrupt file and `.tmp` recovery against a temp directory

## 5. Image loading

- [ ] 5.1 Add `ImageLoader` with its own `WorkerPool(4)`, per-view tokens, `cancel(view)` and a placeholder image in `resources/img/`
- [ ] 5.2 Add the memory LRU (32 MB, halved in applet mode) and disk cache (`cache/images/`, 100 MB); unit-test the LRU logic
- [ ] 5.3 Add a temporary debug row of five Cinemeta posters on the Home placeholder, behind a compile-time flag, to exercise loading and reuse

## 6. Verification

- [ ] 6.1 `ctest --test-dir build_desktop --output-on-failure` passes
- [ ] 6.2 Desktop: run the app and see the debug posters load; restart with the network off and see them come from the disk cache; confirm `~/Library/Application Support/NuvioSwitch/` contents
- [ ] 6.3 Switch: build the `.nro`, run it on hardware or an emulator, confirm the HTTPS fetch and posters work and files appear under `sdmc:/switch/NuvioSwitch/`
- [ ] 6.4 CI passes on both jobs
