## ADDED Requirements

### Requirement: Canonical add-on URLs
The app SHALL identify an add-on by its canonical base URL. To get it, trim whitespace, remove trailing `/`, remove a trailing `/manifest.json` (case-insensitive) from the path part, and keep any `?query` unchanged. Two add-on URLs SHALL be considered the same when their canonical forms are equal ignoring case. The manifest SHALL be fetched from `<path>/manifest.json<?query>`.

#### Scenario: Manifest URL entered
- **WHEN** the user installs `https://example.com/abc/manifest.json`
- **THEN** the add-on is stored as `https://example.com/abc`, and its manifest is fetched from `https://example.com/abc/manifest.json`

#### Scenario: Configured add-on with query
- **WHEN** the user installs `https://example.com/manifest.json?token=X`
- **THEN** the add-on is stored as `https://example.com?token=X`, and its manifest is fetched from `https://example.com/manifest.json?token=X`

#### Scenario: Duplicate ignoring case
- **WHEN** `https://Example.com/abc/` is installed while `https://example.com/abc` already is
- **THEN** no second entry is added, and the user is told the add-on is already installed

### Requirement: Ordered, per-profile add-on list with defaults
The installed add-ons SHALL be an ordered list per profile, with an enabled flag (default on) and an optional custom name per add-on, saved in the `addons` store. The list order SHALL be the add-on priority used by every content feature. When no list has ever been stored, the list SHALL be Cinemeta (`https://v3-cinemeta.strem.io`) followed by OpenSubtitles v3 (`https://opensubtitles-v3.strem.io`).

#### Scenario: First launch
- **WHEN** the app starts with no stored add-on list
- **THEN** the Add-ons tab lists Cinemeta and then OpenSubtitles v3, both enabled

#### Scenario: Empty list stays empty
- **WHEN** the user removes every add-on and restarts the app
- **THEN** the list is still empty, and defaults are not re-added

### Requirement: Manifest parsing
The app SHALL parse Stremio manifests in the forms NuvioTV accepts:
- `resources` entries that are strings, which inherit the manifest's `types` and have no prefixes, or objects `{name, types, idPrefixes}`, where missing `types` inherit the manifest's;
- catalog `extra` entries that are strings or objects `{name, isRequired, options, default, optionsLimit}`, with names trimmed and lowercased, `isRequired` given as a bool, `"true"` or a non-zero number, and blank names dropped;
- the catalog fields `extraSupported`, `extraRequired`, `pageSize` and `showInHome`, recording whether `showInHome` was present at all.

Malformed individual entries SHALL be skipped without rejecting the manifest. A manifest missing `id`, `name` or `version` SHALL be rejected.

#### Scenario: Mixed resources
- **WHEN** a manifest has `types: ["movie","series"]` and `resources: ["catalog", {"name":"stream","types":["movie"],"idPrefixes":["tt"]}]`
- **THEN** the add-on has a `catalog` resource for movie and series with no prefixes, and a `stream` resource for movie with prefix `tt`

#### Scenario: Short-form extras
- **WHEN** a catalog declares `extraSupported: ["search","skip"]` and no `extra`
- **THEN** the catalog is reported as supporting `search` and `skip`

### Requirement: Manifest cache and refresh
Parsed manifests SHALL be cached on disk and used immediately on launch. When a cached manifest is missing for an enabled add-on, it SHALL be fetched right away. Cached manifests SHALL be refreshed in the background at most once every 6 hours: the refresh time is recorded when a refresh starts, even if every fetch fails. When a manifest has never been fetched successfully, the add-on SHALL still appear as a placeholder named after the last path segment of its URL, with no resources or catalogs. A disabled add-on SHALL NOT trigger fetches.

#### Scenario: Offline start
- **WHEN** the app starts without network after having loaded manifests before
- **THEN** all installed add-ons show their cached names, logos and catalogs

#### Scenario: Unreachable new add-on
- **WHEN** an installed add-on's manifest has never loaded and the server is down
- **THEN** the add-on is listed as a placeholder, can be removed, and contributes no catalogs or streams

#### Scenario: Refresh throttled
- **WHEN** a background refresh started 2 hours ago
- **THEN** no new background refresh starts until 6 hours have passed since it started

### Requirement: Display names
An add-on's display name SHALL be its custom name if one is set. Otherwise it SHALL be the manifest name, except that when several add-ons without a custom name share a manifest name, the 2nd, 3rd and later in list order SHALL be shown as "Name (2)", "Name (3)", and so on.

#### Scenario: Two Torrentio instances
- **WHEN** two add-ons both named "Torrentio" are installed, in that order
- **THEN** they are shown as "Torrentio" and "Torrentio (2)"

### Requirement: Capability queries
The app SHALL provide a query returning, in priority order, the enabled add-ons that can serve a resource (`catalog`, `meta`, `stream` or `subtitles`, with `subtitle` accepted as an alias) for a content type and id. An add-on matches when it has a resource with that name whose `types` are empty or contain the type, and whose `idPrefixes` match the id. Resource-level `idPrefixes` are used if non-empty, otherwise the manifest-level ones; with no prefixes at either level, every id matches.

#### Scenario: Prefix match
- **WHEN** stream add-ons are queried for type `movie` and id `tt0111161`, and add-on A declares `idPrefixes: ["tt"]` while add-on B declares `["kitsu:"]`
- **THEN** A is returned and B is not

#### Scenario: Disabled add-on excluded
- **WHEN** a matching add-on is disabled
- **THEN** it is not returned

### Requirement: Add-ons tab
The Add-ons tab SHALL list installed add-ons in priority order, each with logo (or a placeholder), display name, version, description and an enabled indicator. The tab SHALL let the user:
- **install** by entering a URL on the software keyboard;
- **enable or disable** an add-on;
- **move** an add-on up or down;
- **remove** an add-on after confirming;
- **refresh** all manifests.

Installing SHALL accept `http://`, `https://` and `stremio://` (turned into `https://`) URLs. It SHALL fetch the manifest before saving, and SHALL show a success message with the add-on name, or the error, without leaving the tab.

#### Scenario: Install from keyboard
- **WHEN** the user chooses "Install add-on", types `stremio://example.com/manifest.json` and confirms
- **THEN** the manifest at `https://example.com/manifest.json` is fetched, the add-on is appended to the list, and an "Installed <name>" message is shown

#### Scenario: Invalid URL
- **WHEN** the user enters `example.com` (no scheme)
- **THEN** the message "Add-on URL must start with http or https" is shown and nothing is saved

#### Scenario: Manifest fetch fails
- **WHEN** the entered URL returns 404
- **THEN** an error message is shown and nothing is saved

#### Scenario: Reorder
- **WHEN** the user moves the second add-on up
- **THEN** it becomes first, the new order is saved, and later content requests use it

#### Scenario: Remove
- **WHEN** the user removes an add-on and confirms
- **THEN** it disappears from the list, its cached manifest is dropped, and its enabled state and custom name are deleted
