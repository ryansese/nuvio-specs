# NuvioSwitch-0012 Release packaging

## Why

With 0011 the app reaches the v0.1 milestone: browse, play, resume and configure, all offline-local. CI (0001) already builds a `.nro` artifact on every push. But there is no versioned release, no download page, no install guide and no licence, so nobody else can install the app safely or know what version they have. This change turns a CI build into a release users can download and install.

## What Changes

- **Release metadata:** final NACP values (name "NuvioSwitch", author, version from the single CMake version) and a release-quality 256×256 icon from NuvioTV's current logo mark (as agreed for 0001).
- **Versioning rule:**
  - versions are `MAJOR.MINOR.PATCH` and are set only in the root `CMakeLists.txt`;
  - the git tag is `v<version>`;
  - a release fails if the tag already exists or if `CHANGELOG.md` has no section for the version.
- **`CHANGELOG.md`** with a `## <version>` section per release. Its text becomes the GitHub release notes.
- **Release workflow** (`.github/workflows/release.yml`), run by hand with a mode of `draft` or `publish`, in the spirit of NuvioTV's `android-release.yml`. It:
  - reads the version;
  - checks the tag and changelog;
  - builds the `.nro` in the pinned devkitPro container;
  - packages it as `NuvioSwitch.nro` and as `NuvioSwitch-<version>.zip` with the SD-card layout `switch/NuvioSwitch/NuvioSwitch.nro`;
  - writes `SHA256SUMS.txt`;
  - creates the tag and a GitHub Release with those three files attached.
- **README install guide:**
  - requirements: a Switch running custom firmware with the homebrew menu, and an internet connection;
  - copying to `sdmc:/switch/NuvioSwitch/`;
  - launching through title takeover (hold R while starting a game) for full memory, and why album/applet mode is limited;
  - where data is stored, how to update and uninstall, and troubleshooting.
- **`LICENSE`:** GPL-3.0, matching NuvioTV, whose logo, fonts and icons are reused. Third-party notices are listed in `NOTICE.md` (borealis Apache-2.0, libmpv/FFmpeg LGPL/GPL, libcurl, nlohmann/json MIT, qrcodegen MIT, fonts OFL).
- **In-app version** in About (0011) matches the release, and the Switch build shows `0.1.0`. Desktop builds show the commit hash after the version for non-release builds.

## Capabilities

### New Capabilities
- `nuvioswitch-release`: versioning rules, the changelog, the release workflow and its outputs, the install guide, licensing.

### Modified Capabilities
None. `nuvioswitch-build` (0001, not yet archived) gets an `ADDED` requirement for the build-type version suffix and pinned container.

## Impact

- **Depends on:**
  - 0001 (CMake version block, `.nro` target, CI)
  - 0011 (About section shows the version)
  - the whole v0.1 set (0002–0011) being merged before the first `publish` run
- **Code and files (`repos/NuvioSwitch`):**
  - `.github/workflows/release.yml`
  - `CHANGELOG.md`, `LICENSE`, `NOTICE.md`
  - an updated `README.md`
  - `resources/img/icon.jpg` (final)
  - a CMake `GIT_COMMIT` / `NUVIO_RELEASE_BUILD` option
  - `scripts/package-release.sh`
- **GitHub:** creates tags and Releases on `ryansese/NuvioSwitch`. The workflow needs `contents: write`. Nothing is published until someone runs it in `publish` mode.
- **Behaviour references (NuvioTV):**
  - `.github/workflows/android-release.yml` (manual mode, tag and release existence checks)
  - `scripts/release-metadata.sh`
  - `LICENSE` (GPL-3.0)
