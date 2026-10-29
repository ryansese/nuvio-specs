## 1. Versioning

- [ ] 1.1 Move the version block to `cmake/version.cmake` and include it from the root `CMakeLists.txt`
- [ ] 1.2 Add the `NUVIO_RELEASE_BUILD` option and git short-hash detection; generate the display version (`X.Y.Z` or `X.Y.Z-dev+<hash>`) in `version.hpp`; keep the NACP version plain
- [ ] 1.3 Unit test for the display-version formatting function

## 2. Metadata and licensing

- [ ] 2.1 Finalise `resources/img/icon.jpg` (256×256 from NuvioTV's `app_logo_mark.png`) and check NACP name and author
- [ ] 2.2 Add `LICENSE` (GPL-3.0) and `resources/licenses/` with each third-party licence
- [ ] 2.3 Generate `NOTICE.md` from `resources/licenses/`, and point the About licence list (0011) at the same source

## 3. Release workflow

- [ ] 3.1 Add `scripts/package-release.sh`: read the version via `cmake -P`, check the `CHANGELOG.md` section, build the zip with `switch/NuvioSwitch/NuvioSwitch.nro`, write `SHA256SUMS.txt`, extract the notes
- [ ] 3.2 Add `.github/workflows/release.yml`: `workflow_dispatch` with `mode` (draft/publish, default draft), `contents: write`, pinned devkitPro container, tag and release existence checks, build with `NUVIO_RELEASE_BUILD=ON`, `gh release create v<version>` with assets and notes
- [ ] 3.3 Pin the same container tag in `build.yml`
- [ ] 3.4 Add `CHANGELOG.md` with a `## 0.1.0` section summarising 0001–0011

## 4. Install guide

- [ ] 4.1 Rewrite `README.md`: overview and NuvioTV credit, requirements, install from zip or `.nro`, title takeover vs applet mode, data location, update, uninstall, troubleshooting, build-from-source pointer to `CLAUDE.md`

## 5. Verification

- [ ] 5.1 `ctest --test-dir build_desktop` passes; a desktop build shows `-dev+<hash>` in About
- [ ] 5.2 Run `scripts/package-release.sh` locally (or in the container) and check the zip layout and checksums
- [ ] 5.3 Run the workflow in `draft` mode; check the draft release, its assets, the notes, and that a second run for the same version fails
- [ ] 5.4 Switch: install from the draft's zip on a clean SD card following only the README; check the homebrew menu entry (name, author, 0.1.0, icon) and that the app starts and About shows `0.1.0`
- [ ] 5.5 Publish only after the user confirms the draft (`mode: publish`)
