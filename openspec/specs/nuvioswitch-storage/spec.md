## Purpose

TBD: Define the NuvioSwitch per-profile JSON key-value stores, their location on each platform, and atomic writes.

## Requirements

### Requirement: App data location
The app SHALL keep all persistent data under one data folder: `sdmc:/switch/NuvioSwitch/` on Switch, and `~/Library/Application Support/NuvioSwitch/` on macOS. The folder and its subfolders SHALL be created on first use. The HTTP and image caches SHALL live under `cache/` inside it.

#### Scenario: First launch
- **WHEN** the app starts and the data folder doesn't exist
- **THEN** it is created, and the app continues normally

### Requirement: Per-profile JSON stores
Data SHALL be stored as named stores, each one JSON object in its own file. A store belongs either to a profile (`profiles/<profileId>/<store>.json`) or is global (`<store>.json`). Until profiles exist (NuvioSwitch-0013), the app SHALL use profile id `1`. This mirrors NuvioTV's per-profile DataStore files (`data/local/ProfileDataStoreFactory.kt`).

#### Scenario: Read and write a value
- **WHEN** app code writes key `installed_addon_urls_ordered` to the `addons` store of profile 1 and reads it back after restarting the app
- **THEN** the stored value is returned

#### Scenario: Profiles isolated
- **WHEN** profile 1 and profile 2 each write a different value to the same key in the same store
- **THEN** each profile reads back its own value

### Requirement: Atomic, crash-safe writes
Saving a store SHALL write the complete JSON to a temporary file in the same folder and then rename it over the old file, so a crash or power loss leaves either the old or the new contents and never a partial file. Writes SHALL happen off the UI thread, and concurrent saves of the same store SHALL be serialised.

#### Scenario: Interrupted write
- **WHEN** the app is killed while a store is being saved
- **THEN** on next launch the store contains either the previous or the new data, and loads without error

### Requirement: Corrupt store recovery
If a store file can't be parsed as JSON, the app SHALL rename it to `<store>.json.corrupt`, log the problem, and continue with an empty store.

#### Scenario: Corrupt file
- **WHEN** `addons.json` contains invalid JSON at startup
- **THEN** it is renamed to `addons.json.corrupt`, the app starts, and the store reads as empty
