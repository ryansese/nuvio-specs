## 1. libmpv dependency

- [ ] 1.1 Check the available libmpv packages for Switch (devkitPro portlib names, hwdec support) and for desktop (Homebrew); benchmark 1080p H.264 and HEVC on Switch and record the chosen build in NuvioSwitch's `CLAUDE.md` after the user confirms it
- [x] 1.2 Link libmpv in `CMakeLists.txt` via `pkg-config` on both platforms, with a clear error if it's missing
- [x] 1.3 Update CI: install mpv on the macOS runner and the libmpv portlib in the devkitPro job

## 2. Player core

- [x] 2.1 Add `src/player/player_rules.{hpp,cpp}`: header fields, loadfile arguments, seek step, subtitle URL, track-list parsing, time formatting
- [x] 2.2 Unit tests for each helper, using the spec's examples (header escaping and sorting, `start=+600.000`, repeat thresholds, subtitle URL with hash and size, `tv` → `series`)
- [x] 2.3 Add `src/player/mpv_core.{hpp,cpp}`: create the handle, set the shared and platform options, create the OpenGL render context, observe properties, handle wakeup through `brls::sync`, provide typed command and property helpers, tear down in the right order
- [x] 2.4 Add `src/player/player_view.{hpp,cpp}`: a borealis view that renders the mpv frame with GL state saved and restored around the call
- [x] 2.5 Add `src/player/subtitle_repository.{hpp,cpp}`: select add-ons by resource, type and prefix; fetch in parallel with 20 s timeouts; merge with the stream's own subtitles

## 3. Player screen

- [x] 3.1 Add `resources/xml/activity/player.xml` and `src/activity/player_activity.{hpp,cpp}`, opened from the stream picker with a `PlaybackRequest`
- [x] 3.2 Loading spinner until the first frame; error overlay with "Try another stream" and "Back"
- [x] 3.3 Controls overlay: title and episode line, seek bar with buffered range, elapsed and remaining time, Play/Pause, Subtitles, Audio, Sources; 4 s auto-hide while playing
- [x] 3.4 Gamepad input: A play/pause, Left/Right accelerated seek with a repeat counter, ZL/ZR ±10 s, Up/Down/X show controls, B hide or exit
- [x] 3.5 Audio and Subtitles side panels from `track-list` plus external subtitles; "Off"; reset `aid`/`sid` to `auto` on each new file
- [x] 3.6 Sources returns to the stream picker with its results intact
- [x] 3.7 Add the i18n strings

## 4. Verification

- [x] 4.1 `ctest --test-dir build_desktop` passes
- [x] 4.2 Desktop: play an HTTP stream from a real add-on, a stream with `proxyHeaders`, and an episode with OpenSubtitles results; check seeking, held seek acceleration, audio and subtitle switching, Sources, and the error overlay with a dead URL
- [ ] 4.3 Switch: build the `.nro` and play the same streams on hardware via `nxlink -s`, handheld and docked; check smoothness at 720p and 1080p, subtitles, and exit back to the picker
- [ ] 4.4 Switch: start and exit playback ten times and confirm no crash and stable memory
