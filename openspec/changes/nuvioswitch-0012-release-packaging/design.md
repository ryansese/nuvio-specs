## Context

0001 set up the CMake version block, the `NuvioSwitch.nro` target (`nacptool --create <name> <author> <version>` and `elf2nro --icon --nacp --romfsdir`, following borealis's root `CMakeLists.txt`), and a `build.yml` CI that uploads the `.nro` as an artifact.

NuvioTV releases through a manually dispatched workflow, `.github/workflows/android-release.yml`. It has preview, draft and publish modes, checks that the tag and release don't already exist, and generates notes with `scripts/release-metadata.sh` and `scripts/generate-release-notes.sh`. NuvioTV is GPL-3.0 (`LICENSE`), and NuvioSwitch reuses its logo, fonts and icons.

## Goals / Non-Goals

**Goals:**
- One repeatable command (a workflow dispatch) that produces a verified, versioned GitHub Release.
- Clear install instructions for homebrew users.
- Correct licensing before anything is distributed.

**Non-Goals:**
- An in-app updater: NuvioTV's `app/src/full/java/com/nuvio/tv/updater/` is Android-only, and it's out of scope for v0.1.
- Distribution through the Homebrew App Store or other stores; this can come later.
- Automatic release notes from commit history (NuvioTV's Python generator). A hand-written changelog is enough at this pace.
- Desktop release builds. Desktop stays a developer build.

## Decisions

**1. Manual dispatch with `draft`/`publish` modes rather than releasing on tag push.**
- This mirrors NuvioTV, and lets the maintainer check a draft before it goes public.
- The workflow creates the tag itself, so the version in `CMakeLists.txt` is the single source.
- Alternative: release on pushing a `v*` tag. Rejected, because it allows the tag and the CMake version to disagree.

**2. Version extraction.**
- `scripts/package-release.sh` reads `VERSION_MAJOR`/`MINOR`/`ALTER` with `cmake -P` (a tiny script that includes the version block), not with grep.
- The version block moves to `cmake/version.cmake`, which both the root `CMakeLists.txt` and the script include.

**3. Release vs dev version.**
- `NUVIO_RELEASE_BUILD` controls the display suffix in the generated `version.hpp`; `git rev-parse --short HEAD` is run at configure time.
- The NACP always gets the plain version, because the homebrew menu and update tools compare it.

**4. Packaging.**
- The zip layout `switch/NuvioSwitch/NuvioSwitch.nro` matches where users put homebrew and where the app keeps its data (0002's storage root), so extracting to the SD card root just works.
- `SHA256SUMS.txt` uses `sha256sum` format.
- Release creation uses `gh release create` with `--draft` in draft mode. No third-party actions are needed beyond `actions/checkout`.

**5. Pinned container.**
- `devkitpro/devkita64:<dated tag>` is pinned in both `build.yml` and `release.yml`.
- Bumping it is a deliberate commit, which keeps rebuilds reproducible.

**6. Licence: GPL-3.0.**
- NuvioSwitch reuses GPL-3.0 assets from NuvioTV, and it ships libmpv/FFmpeg, whose builds in devkitPro portlibs include GPL parts.
- GPL-3.0 is compatible with borealis (Apache-2.0) and the MIT components.
- `NOTICE.md` and the About licence list are generated from one source, `resources/licenses/`, to keep them in step.

**7. Icon.** The 256×256 JPEG from NuvioTV's current `app_logo_mark.png` (decided in 0001) is regenerated at final quality and checked in the homebrew menu.

## Risks / Trade-offs

- [GPL obligations: the source for each release must be available] → The release links the tag, and the source is public on GitHub.
- [devkitPro portlib versions of mpv/FFmpeg change under a pinned container only when the image changes] → This is the intent of pinning. Record the container tag in the release notes.
- [Users launching in applet mode hit memory limits during playback] → The README explains title takeover. Detecting applet mode and showing an in-app warning is a candidate follow-up.
- [Accidental public release] → The default mode is `draft`, and `publish` must be chosen explicitly.

## Open Questions

- Is GPL-3.0 the licence the user wants for NuvioSwitch? It follows from reusing NuvioTV assets. A more permissive licence would mean replacing those assets.
