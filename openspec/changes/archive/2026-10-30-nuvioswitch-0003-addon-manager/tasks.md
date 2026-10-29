## 1. Models and pure logic

- [x] 1.1 Add `src/domain/addon.hpp` with `Addon`, `AddonResource`, `CatalogDescriptor` (with `hasExplicitShowInHome`), `CatalogExtra`, `AddonBehaviorHints`, and `supportsExtra(name)` on catalogs
- [x] 1.2 Add `addon_url` (`canonicalize`, `normalizeForCompare`, `manifestUrl`, `normalizeInstallInput`) with unit tests for trailing slash, `/manifest.json` in any case, query strings and `stremio://`
- [x] 1.3 Add `manifest_parser` with unit tests against fixture manifests in `tests/fixtures/` (Cinemeta, OpenSubtitles v3, configurable add-on, malformed entries, missing id)
- [x] 1.4 Add `addon_display::applyDisplayNames` with unit tests for duplicates and custom names
- [x] 1.5 Add `addon_query` (`supportsResource`, `addonsFor`) with unit tests for type matching, resource-level vs manifest-level prefixes, `subtitle` alias and disabled add-ons

## 2. Repository

- [x] 2.1 Add `AddonRepository` backed by the `addons` store: load with defaults, `add`, `remove`, `setEnabled`, `move(index, delta)`, change subscription
- [x] 2.2 Add the manifest cache in the global `addon_manifest_cache` store (raw JSON plus `fetchedAt`, `lastRefreshAttemptAt`)
- [x] 2.3 Implement fetch policy: immediate fetch for cache misses, 6-hour background refresh recorded at start, forced refresh, placeholders on failure, no fetches for disabled add-ons
- [x] 2.4 Create the repository at startup in `main.cpp` and expose it to tabs

## 3. Add-ons tab

- [x] 3.1 Add `resources/xml/tabs/addons.xml` and `AddonsTab`, replacing the 0001 placeholder; add i18n strings (install, refresh, enable, disable, move up, move down, remove, confirm, errors, "already installed", "Installed <name>")
- [x] 3.2 Add `AddonCell` (logo via `ImageLoader`, name, version, description, disabled badge, placeholder note)
- [x] 3.3 Add the install flow: IME text input, validation, spinner, fetch, save, notification
- [x] 3.4 Add per-add-on actions (A menu, X toggle, L/R move, remove confirmation) and footer hints
- [x] 3.5 Add the Refresh action

## 4. Verification

- [x] 4.1 `ctest --test-dir build_desktop --output-on-failure` passes
- [x] 4.2 Desktop: first launch shows Cinemeta and OpenSubtitles; install a third add-on by URL, reorder, disable, remove; restart and check everything persisted; restart offline and see cached manifests (verified by the user on macOS)
- [x] 4.3 Switch: same flow on hardware or emulator with swkbd; check `sdmc:/switch/NuvioSwitch/profiles/1/addons.json` (verified by the user on Switch)
- [x] 4.4 CI passes on both jobs (run 40935824892 on f3a98fb: both jobs green)
