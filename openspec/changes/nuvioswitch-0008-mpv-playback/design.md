## Context

NuvioTV plays streams with Media3 by default, with libmpv as a second engine. The mpv path is:

- **`ui/screens/player/NuvioMpvSurfaceView.kt`**
  - `initOptions` (~line 641) sets the options;
  - `applyHeaders` (~676) sorts and escapes the headers;
  - `loadFileWithOptions` (~113) puts the options in the fourth argument, after index `-1`;
  - `sub-add <url> cached <title> <lang>` (~516) adds subtitles;
  - track reading uses `track-list/N/*` (~444, ~549).
- **`PlayerRuntimeControllerMpv.kt`**: drives the view and polls position and duration.
- **`PlayerScrubRates.kt`**: seek steps by repeat count.
- **`data/repository/SubtitleRepositoryImpl.kt`**: add-on subtitles, with a 20 s timeout per add-on, `tv` treated as `series`, and `buildExtraParams`.

The Android-only parts don't apply: `vo=gpu` with `gpu-context=android`, `ao=audiotrack`, `mediacodec` hwdec, the CA bundle in app files, and the Hi10P fallback.

Borealis renders with NanoVG on OpenGL (desktop GLFW; Switch EGL/OpenGL through mesa/nouveau) or optionally deko3d. Switchfin and wiliwili are borealis apps that play video with libmpv's render API. Upstream libmpv's render API offers OpenGL and software rendering. A deko3d render path exists only in patched mpv builds used by those projects.

## Goals / Non-Goals

**Goals:**
- Reliable playback of add-on HTTP streams on Switch (handheld and docked) and macOS, with headers, seeking and subtitles.
- A small `MpvCore` wrapper that 0009 (progress) and 0010 (autoplay, next episode, skip intro) build on, through properties and events rather than reaching into mpv directly.

**Non-Goals:**
- deko3d rendering, unless the OpenGL path proves too slow (see Risks).
- Picture settings, subtitle styling, audio delay, speed and aspect, which come with 0010 and 0011.
- Saving progress and resume, which 0009 adds through `MpvCore` events.

## Decisions

**1. Render path: libmpv render API with `MPV_RENDER_API_TYPE_OPENGL` on both platforms.**
- borealis already runs on OpenGL by default. The mpv frame is rendered into an FBO-backed texture and drawn by a `PlayerView` in borealis's draw pass, or into the default framebuffer before NanoVG draws the controls.
- `get_proc_address` comes from glfw on desktop and EGL on Switch.
- Rendering happens on the UI thread when `mpv_render_context_set_update_callback` signals a new frame (the callback only flags it and wakes the loop). This keeps all GL calls on one thread.
- Alternative: deko3d through a patched mpv, as Switchfin and wiliwili offer. Deferred: it needs a custom mpv/ffmpeg build and borealis's `USE_DEKO3D`, and it's only worth doing if OpenGL can't keep 1080p smooth.

**2. libmpv packages.**
- Desktop: `brew install mpv`, found with `pkg_check_modules(MPV REQUIRED mpv)`.
- Switch: the devkitPro libmpv portlib and its ffmpeg dependency via `pkg-config` in the devkitPro container.
- The first apply task checks which portlib names exist and whether hardware decoding is available. If the stock portlib can't decode 1080p H.264/HEVC smoothly, use the prebuilt mpv/ffmpeg that Switchfin and wiliwili publish for Switch, pinned in CI. Record that choice in NuvioSwitch's `CLAUDE.md`.

**3. Platform options.**
- Desktop: `vo=libmpv`, `hwdec=auto-safe`, default `ao`.
- Switch: `vo=libmpv`, plus `hwdec` and `ao` set to whatever the chosen mpv build supports (`hwdec=auto` if a Tegra hwdec exists, otherwise `no`, and `ao=hos` or the build's default).
- Everything else comes from the NuvioTV list in the spec. `tls-verify` follows 0002's networking choice: NuvioTV verifies TLS for media but not for add-on requests. On Switch the CA bundle comes from the system (`ssl` service via mbedtls in the build), checked during apply.

**4. `MpvCore` events instead of polling.**
- `mpv_observe_property` watches `time-pos`, `duration`, `pause`, `paused-for-cache`, `demuxer-cache-time`, `track-list` and `eof-reached`.
- `mpv_set_wakeup_callback` posts `brls::sync` to drain `mpv_wait_event(0)` on the UI thread.
- `MpvCore` exposes simple callbacks (`onPosition`, `onTracks`, `onEnd`, `onError` from `MPV_EVENT_END_FILE` with `MPV_END_FILE_REASON_ERROR`). NuvioTV polls because of its Android bridge; events are simpler here.

**5. Pure helpers in `src/player/player_rules.{hpp,cpp}`, unit-tested:**
- `headerFields(map)`;
- `loadfileArgs(url, startMs)`;
- `seekStepMs(repeat)`;
- `subtitleUrl(addon, type, id, videoId, hash, size, filename)`;
- `parseTrackList(json)`, which reads `track-list` as a node or JSON string;
- `formatTime`.

**6. Subtitle repository.** `SubtitleRepository::fetchAll(request, onDone)` uses 0002's pool with a 20 s timeout per request, and is started when the player opens so the panel is ready. Add-on subtitles are only `sub-add`ed when chosen, which avoids downloading dozens of files.

**7. UI.**
- `PlayerActivity` contains `PlayerView` (full screen), an overlay `Box` for the controls (bottom gradient, title, seek bar from borealis `Slider` or a custom bar, time labels, buttons), side panels (right-aligned, 360 px wide like NuvioTV's `SubtitleSelectionOverlay`) and an error overlay.
- Input uses `registerAction` with `allowRepeating=true` for Left/Right. A repeat counter resets on release (no repeat event within 250 ms) and feeds `seekStepMs`.
- The 4-second auto-hide timer is a `brls::Ticking` or a delayed `brls::sync` cancelled on input.

**8. Teardown order on exit:** `stop` command, `mpv_render_context_free`, `mpv_terminate_destroy`, then pop the activity. This prevents GL calls on a freed context, the most common crash in borealis + mpv apps.

## Risks / Trade-offs

- [Stock Switch libmpv may decode only in software, so 1080p HEVC stutters] → Benchmark in the first apply task. Fallbacks, in order: a hwdec-enabled build from the Switchfin/wiliwili ecosystem; `vd-lavc-threads` tuning and `profile=fast`; deko3d as a later spec.
- [OpenGL interop between NanoVG and mpv (state leaks like blend mode or bound FBO)] → Save and restore GL state around `mpv_render_context_render`, as wiliwili does. Draw mpv first, then NanoVG controls.
- [Memory on Switch (applet mode has about 400 MB)] → Keep demuxer caches at 64 MiB, as NuvioTV does. Document that NuvioSwitch should run in title (full memory) mode via a game override. Revisit cache sizes if out-of-memory errors show up.
- [CI needs mpv in both environments] → Pin the Homebrew mpv version in CI and install the portlib explicitly in the devkitPro job.

## Open Questions

- Which libmpv build is used on Switch (stock portlib or a hwdec-enabled third-party build) is decided by the benchmark in task 1.1; the user should confirm the choice before it's locked into CI.
