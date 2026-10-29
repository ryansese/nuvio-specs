## ADDED Requirements

### Requirement: Versioning
The app version SHALL be a `MAJOR.MINOR.PATCH` value set only in the root `CMakeLists.txt`. Each release SHALL be tagged `v<version>` on the commit it was built from. A version SHALL NOT be released twice.

#### Scenario: Duplicate version
- **WHEN** the release workflow runs for version `0.1.0` and the tag `v0.1.0` already exists
- **THEN** the workflow fails before building, with a message to bump the version

### Requirement: Changelog
`CHANGELOG.md` SHALL have one `## <version>` section per release, newest first, listing user-visible changes. The section for the version being released SHALL be used as the GitHub release notes.

#### Scenario: Missing changelog entry
- **WHEN** the release workflow runs for `0.2.0` and `CHANGELOG.md` has no `## 0.2.0` section
- **THEN** the workflow fails with a message naming the missing section

### Requirement: Release workflow
The repository SHALL have a manually triggered release workflow with a `mode` input of `draft` or `publish`. It SHALL:
1. Read the version from `CMakeLists.txt`.
2. Check the tag and changelog rules.
3. Build the Switch target in the pinned devkitPro container with `NUVIO_RELEASE_BUILD=ON`.
4. Create `NuvioSwitch.nro`, `NuvioSwitch-<version>.zip` (containing `switch/NuvioSwitch/NuvioSwitch.nro`) and `SHA256SUMS.txt` covering both.
5. Create the tag and a GitHub Release with those three assets and the changelog section as notes.

The release SHALL be a draft in `draft` mode, and public in `publish` mode.

#### Scenario: Draft release
- **WHEN** the maintainer runs the workflow in `draft` mode for `0.1.0`
- **THEN** a draft release `v0.1.0` exists with the `.nro`, the zip and `SHA256SUMS.txt` attached, and nothing is visible publicly

#### Scenario: Checksums match
- **WHEN** a user downloads `NuvioSwitch.nro` from the release and computes its SHA-256
- **THEN** it matches the entry in `SHA256SUMS.txt`

### Requirement: NACP metadata and icon
The released `.nro` SHALL carry:
- the name "NuvioSwitch", the author "Ryan Sese", and the release version;
- the 256×256 JPEG icon made from NuvioTV's current logo mark.

The homebrew menu SHALL show this name, author, version and icon.

#### Scenario: Homebrew menu entry
- **WHEN** the user opens the homebrew menu with `NuvioSwitch.nro` from release `0.1.0` installed
- **THEN** it shows "NuvioSwitch", "Ryan Sese", "0.1.0" and the Nuvio logo

### Requirement: Install guide
`README.md` SHALL explain:
- what NuvioSwitch is and its relation to NuvioTV;
- the requirements: custom firmware with the homebrew menu, internet;
- installing: download the zip, extract it to the SD card root, or copy the `.nro` to `sdmc:/switch/NuvioSwitch/`;
- launching with title takeover (hold R while starting a game) for full memory, and the limits of applet mode;
- where app data is stored (`sdmc:/switch/NuvioSwitch/`), how to update (replace the `.nro`, data is kept) and how to uninstall;
- basic troubleshooting.

#### Scenario: Fresh user install
- **WHEN** a user follows the README with a fresh SD card and the release zip
- **THEN** NuvioSwitch appears in the homebrew menu and starts

### Requirement: Licensing
The repository SHALL contain a GPL-3.0 `LICENSE` and a `NOTICE.md` listing every third-party component in the release with its licence. The About screen's licence list SHALL match `NOTICE.md`. The components are borealis, libmpv, FFmpeg, libcurl, nlohmann/json, qrcodegen, and the fonts and icons taken from NuvioTV.

#### Scenario: Licence present in release
- **WHEN** a user downloads the release zip
- **THEN** the repository's `LICENSE` and `NOTICE.md` are linked from the release notes
