## Context

0002 provides `HttpClient` (with `permissiveTls`), `Store` (per-profile JSON), the JSON helpers and `ImageLoader`. The Add-ons tab is a placeholder from 0001.

In NuvioTV:
- **Install state** is in `data/local/AddonPreferences.kt`: `installed_addon_urls_ordered` (JSON list), `installed_addon_enabled_states` (JSON map URL→bool) and `addon_user_set_names` (JSON map URL→name). It falls back to `getDefaultAddons()` when nothing is stored.
- **Behaviour** is in `data/repository/AddonRepositoryImpl.kt`:
  - `canonicalizeUrl` and `fetchAddon` (manifest URL `basePath/manifest.json + query`);
  - the manifest cache in SharedPreferences `addon_manifest_cache`/`manifests_v2`;
  - a refresh at most every 6 h (`MANIFEST_CACHE_TTL_MS`), whose time is recorded when it starts;
  - `placeholderAddon` and `applyDisplayNames`.
- **Parsing** is in `data/mapper/AddonMapper.kt`.
- **Install, validation and reorder** are in `ui/screens/addon/AddonManagerViewModel.kt`.
- **Resource matching** is in `core/streams/PlaybackAvailability.kt` (`supportsStreamResource`).

## Goals / Non-Goals

**Goals:**
- A single `AddonRepository` that every content feature asks for "which add-ons, in what order, can do X", matching NuvioTV's results for the same manifests.
- Storage keys and values that match NuvioTV's, so account sync (0014) can push and pull the same data.
- A usable gamepad UI for managing add-ons without a phone.

**Non-Goals:**
- NuvioTV's "Manage from phone" QR and local web server (`core/server/`), and the Essential-mode add-on setup screen.
- Catalog ordering and disabling individual catalogs (`CatalogOrderScreen.kt`) and Collections. Home uses add-on order × manifest catalog order (0004).
- Profiles that share profile 1's add-ons (`usesPrimaryAddons`). There is only profile 1 until 0013.
- Renaming add-ons from the UI. The custom-name field is stored and respected, but setting it waits for a later change or comes from sync.
- Plugins (QuickJS) and CloudStream, which are out of scope for the port.

## Decisions

**1. Pure core, thin UI.** Layout:
- `src/domain/addon.hpp`: `Addon`, `AddonResource`, `CatalogDescriptor`, `CatalogExtra`, `AddonBehaviorHints`. The fields follow `domain/model/Addon.kt`, including `hasExplicitShowInHome`.
- `src/data/addons/addon_url.{hpp,cpp}`: `canonicalize`, `normalizeForCompare`, `manifestUrl`, and `normalizeInstallInput`, which handles the `stremio://` rewrite and the scheme check. All pure.
- `src/data/addons/manifest_parser.{hpp,cpp}`: JSON to `Addon`. Pure, using the 0002 JSON helpers.
- `src/data/addons/addon_display.{hpp,cpp}`: `applyDisplayNames`. Pure.
- `src/data/addons/addon_query.{hpp,cpp}`: `supportsResource(addon, resource, type, id)` and `addonsFor(...)`. Pure. `catalog` queries ignore the id.
- `src/data/addons/addon_repository.{hpp,cpp}`: state, persistence, fetches and refresh. It's UI-thread owned. Network results arrive through `brls::sync`, so no locks are needed.
- `src/ui/addons/`: `AddonsTab`, `AddonCell` and `InstallAddonFlow`.

Everything except the repository and UI is compiled into `NuvioSwitchTests`, which test against fixture manifests copied from real add-ons (Cinemeta, OpenSubtitles v3, and a configurable add-on with a query) into `tests/fixtures/`.

**2. Change notification is an observer list, not reactive streams.** `AddonRepository::subscribe(cb)` returns a token, and callbacks fire on the UI thread after any change (install, remove, reorder, toggle, or a manifest arriving). Home (0004) subscribes and rebuilds when the add-on set changes.
- Alternative: a port of Kotlin Flow. Rejected; that's overkill here.

**3. Storage format mirrors NuvioTV.**
- In store `addons`:
  - `installed_addon_urls_ordered` is a JSON array of canonical URLs, not a JSON string inside JSON, since our store is already JSON;
  - `installed_addon_enabled_states` is an object;
  - `addon_user_set_names` is an object.
- The absence of `installed_addon_urls_ordered` means "never stored", so defaults apply. An empty array means the user removed everything.
- The manifest cache is a global store, `addon_manifest_cache`, with `manifests_v2` mapping canonical URL to the raw manifest JSON plus `fetchedAt`, and `lastRefreshAttemptAt`. It stores raw JSON rather than our parsed struct, so parser fixes apply to cached data without a migration.

**4. Fetch policy.**
- Manifest requests use `permissiveTls=true` and `cachePolicy=NoCache`, because the manifest cache handles freshness.
- On startup:
  - load the cache and publish the list immediately;
  - fetch any enabled add-on with no cached manifest;
  - if `now - lastRefreshAttemptAt > 6h`, set `lastRefreshAttemptAt = now` first, then refresh all enabled add-ons in parallel.
- A refresh replaces a cached manifest only when the new raw JSON differs. Failures leave the old cache in place.
- "Refresh" in the tab forces a refresh of all add-ons and updates `lastRefreshAttemptAt`.

**5. Install flow.**
1. The user presses "Install add-on" (the first item in the tab), and `brls::Application::getImeManager()->openForText(...)` opens swkbd on Switch or the desktop text dialog.
2. `normalizeInstallInput` validates the URL.
3. A blocking-overlay spinner shows while the manifest is fetched.
4. On success the add-on is saved at the end of the list with `enabled=true`, and a toast-style notification shows (`brls::Application::notify`).
5. If the canonical URL is already installed, the app shows "already installed" without fetching.

This is a small deviation: NuvioTV fetches first and then silently no-ops. The message is clearer.

**6. List UI.**
- A vertical `RecyclerFrame`, one `AddonCell` per add-on: logo via `ImageLoader`, name, "v<version>", a two-line description, and an "Off" badge when disabled.
- The "Install add-on" and "Refresh" cells sit above the list.
- Per-add-on actions open from **A** as a `brls::Dropdown` with Enable/Disable, Move up, Move down and Remove. Remove shows a `brls::Dialog` confirmation.
- Shortcuts: **X** toggles the add-on, and **L/R** move it up or down. Each shortcut also appears in the footer hints.
- Alternative: drag-to-reorder. Rejected; it's awkward with a gamepad. NuvioTV also uses up/down buttons.

**7. Placeholder add-ons.** These are built the same way as NuvioTV's `placeholderAddon`: id and baseUrl are the canonical URL, the name is the custom name or the last path segment before `?`, and resources and catalogs are empty. They're shown with a "Couldn't load manifest" note.

## Risks / Trade-offs

- [Big manifests (some add-ons list hundreds of catalogs) make parsing and storage slow on the Switch CPU] → Parse on the worker thread before posting to the UI. Measure with the largest add-on we test, and only cache raw JSON.
- [Configurable add-ons keep secrets (tokens) in the query string] → Those secrets are stored in plain JSON on the SD card. NuvioTV has the same exposure. Never log full add-on URLs; log the host only.
- [Profile 1 is hard-coded] → `AddonRepository` takes its profile from `ProfileContext`, so 0013 only has to switch the store.

## Open Questions

None.
