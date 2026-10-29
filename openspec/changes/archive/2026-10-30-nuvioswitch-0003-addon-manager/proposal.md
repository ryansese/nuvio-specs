# NuvioSwitch-0003 Add-on manager

## Why

NuvioSwitch has no content of its own. Like NuvioTV, it gets all of its catalogs, metadata, streams and subtitles from Stremio-compatible add-ons. Every content feature (Home, Search, Details, Streams) first needs to know which add-ons are installed, in what priority order, and what each one's manifest says it can do. This change adds that list, plus a screen to manage it.

## What Changes

- **Add-on list.** An ordered, per-profile list of installed add-ons, stored with the 0002 storage layer.
  - Each entry has its URL, an enabled flag and an optional custom name. The list order is the add-on priority.
  - URLs are canonicalised and compared the way NuvioTV does: trim, drop a trailing `/`, strip `/manifest.json`, keep `?query`, case-insensitive duplicate check.
  - First launch with no stored list installs Cinemeta (`https://v3-cinemeta.strem.io`) and OpenSubtitles v3 (`https://opensubtitles-v3.strem.io`).
- **Manifest parsing** of the Stremio manifest into C++ models: id, name, version, description, logo, types, `idPrefixes`, `resources` (string or object form), `catalogs` (with `extra` in string or object form, `extraSupported`, `extraRequired`, `pageSize`, `showInHome`) and `behaviorHints`.
- **Manifest cache.**
  - Persisted between launches, and used immediately at startup.
  - Background refresh at most every 6 hours.
  - A missing manifest is fetched straight away.
  - An add-on whose manifest can't be fetched still shows up, as a placeholder with no resources, so it can be removed.
- **Display names.** When several add-ons share a name, the 2nd, 3rd and later ones are shown as "Name (2)", "Name (3)", and so on. A custom name always wins.
- **Capability queries** for later changes: "add-ons that serve resource R for type T and id X", in priority order, with `idPrefixes` matching at resource or manifest level.
- **Add-ons tab.** It replaces the 0001 placeholder:
  - install from a URL typed on the software keyboard (`stremio://` accepted and turned into `https://`);
  - a list of installed add-ons with logo, name, version and description;
  - per add-on: enable/disable, move up/down, remove (with confirmation);
  - a "Refresh" action that re-fetches all manifests.

## Capabilities

### New Capabilities
- `nuvioswitch-addons`: the installed add-on list, URL rules, defaults, manifest parsing and caching, display names, capability queries, and the Add-ons tab.

### Modified Capabilities
None. The Add-ons tab placeholder from `nuvioswitch-app-shell` is replaced by real content, which the shell's placeholder requirement already allows ("until its feature change lands").

## Impact

- **Depends on:**
  - NuvioSwitch-0001: app shell and the Add-ons tab slot;
  - NuvioSwitch-0002: `HttpClient` with `permissiveTls`, the `Store`, the JSON helpers and `ImageLoader` for add-on logos.
- **Code:** new `src/domain/addon.hpp` (models), `src/data/addons/` (manifest parser, URL rules, repository, manifest cache) and `src/ui/addons/` (tab, list cell, install dialog), plus `resources/xml/tabs/addons.xml`, i18n strings and unit tests.
- **Storage:**
  - store `addons` (per profile), with keys matching NuvioTV's names so 0014 sync maps cleanly: `installed_addon_urls_ordered`, `installed_addon_enabled_states` and `addon_user_set_names`;
  - global store `addon_manifest_cache` (`manifests_v2`, map from URL to parsed manifest).
- **Used by:** 0004 Home, 0005 Search, 0006 Details, 0007 Streams and 0014 Account sync.
- **NuvioTV references:**
  - `data/repository/AddonRepositoryImpl.kt`: `canonicalizeUrl`, `fetchAddon`, the cache TTL, `placeholderAddon`, `applyDisplayNames`;
  - `data/local/AddonPreferences.kt`: keys, add/remove, `getDefaultAddons`;
  - `data/remote/dto/AddonManifestDto.kt` and `data/mapper/AddonMapper.kt`: parsing;
  - `domain/model/Addon.kt`;
  - `ui/screens/addon/AddonManagerViewModel.kt`: install validation, reorder;
  - `core/streams/PlaybackAvailability.kt`: resource and prefix matching.
