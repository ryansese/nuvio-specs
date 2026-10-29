# NuvioSwitch-0008 mpv playback

## Why

The stream picker (0007) produces a `PlaybackRequest`, but the app can't play video yet. NuvioTV already has a full libmpv playback path. Its options, header handling, external subtitles and track selection work with Stremio add-on streams, so porting that behaviour to libmpv on Switch and desktop gives the shortest route to playback that works. This change completes the core loop: browse → details → stream → play.

## What Changes

- Add libmpv as a dependency: the devkitPro portlib on Switch, Homebrew `mpv` on macOS.
- Embed it in borealis through mpv's render API (OpenGL, matching borealis's default backend), following how Switchfin and wiliwili integrate mpv with borealis.
- **Player screen:** a full-screen activity opened with a `PlaybackRequest`. It loads the URL with the stream's request headers through `http-header-fields`:
  - entries are sorted;
  - `\` and `,` are escaped;
  - `Range` is already removed by 0007.
- **mpv options** are ported from NuvioTV's `NuvioMpvSurfaceView.initOptions` where they apply outside Android:
  - `keep-open=yes`, 64 MiB `demuxer-max-bytes` and `demuxer-max-back-bytes`;
  - the subtitle defaults (`sub-ass-override=no`, `sub-codepage=auto:utf-8`, `sub-use-margins`, `sub-ass-force-margins`);
  - `volume-max`, `save-position-on-quit=no`;
  - a Nuvio user agent;
  - platform-specific `hwdec` and `ao`.
- **Start position:** when a start position is given, use `loadfile <url> replace -1 start=+<sec>`; the options go in the fourth argument, after the index (`NuvioMpvSurfaceView.loadFileWithOptions`). 0009 supplies the resume position, so this change only accepts it.
- **External subtitles:**
  - the stream's own subtitles plus subtitles from every enabled add-on with a `subtitles` resource for that type and ID;
  - URL `{base}/subtitles/{type}/{id}[/videoHash=..&videoSize=..&filename=..].json`, with `tv` mapped to `series` and the video ID used for series;
  - 20 s timeout per add-on (`data/repository/SubtitleRepositoryImpl.kt`);
  - loaded on demand with `sub-add <url> cached <title> <lang>`.
- **Audio and subtitle pickers:** side panels built from mpv's `track-list`, including external subtitles; plus a subtitle "Off" entry.
- **Controls, following `ui/screens/player/PlayerScreen.kt`:**
  - with controls hidden: A toggles play/pause; Left/Right seek, with the step growing the longer the button is held (10/20/30/60 s at 0/3/8/15 repeats, `PlayerScrubRates.kt`); Up/Down/X show the controls;
  - with controls visible: a bottom bar with title, seek bar, elapsed and remaining time, Play/Pause, Subtitles, Audio and Sources;
  - B hides the controls, or closes the player when they're already hidden;
  - ZL/ZR seek ±10 s at any time.
- **Sources** returns to the stream picker so the user can pick another stream. **Episodes** is deferred to 0010.
- **Lifecycle:**
  - a loading spinner until the first frame;
  - an error overlay with Back and "Try another stream" when mpv reports a load failure;
  - the player shuts down cleanly on exit;
  - progress is shown on screen only; storing progress is 0009.

## Capabilities

### New Capabilities
- `nuvioswitch-playback`: playing a `PlaybackRequest` with libmpv: options, headers, start position, external subtitles, track selection, gamepad controls, errors and teardown.

### Modified Capabilities
None.

## Impact

- **Code (NuvioSwitch):**
  - `src/player/` holds an `MpvCore` wrapper (handle and render context, event loop, typed property and command helpers), a `PlayerView` (a borealis view that draws the mpv frame) and pure helpers (header field string, loadfile arguments, seek step, subtitle URL, track-list parsing);
  - `src/player/subtitle_repository.{hpp,cpp}`;
  - `src/activity/player_activity.{hpp,cpp}`;
  - `resources/xml/activity/player.xml` and side panel XML;
  - i18n strings.
- **Build:** link libmpv via `pkg-config` on desktop (`brew install mpv`) and the Switch portlib on Switch. Update CI (0001) to install mpv on the macOS runner and the libmpv portlib in the devkitPro container.
- **Reuses from earlier changes:** `PlaybackRequest` and the stream picker (0007); the add-on model (0003); HTTP and the URL encoder (0002, 0007).
- **Depends on:** 0001, 0002, 0003, 0007.
- **Out of scope:**
  - saving progress, resume and watched state (0009);
  - autoplay, next episode, skip intro, preferred languages and subtitle styling (0010);
  - player settings UI (0011);
  - Dolby Vision, frame-rate switching and the ExoPlayer engine;
  - computing the OpenSubtitles hash for streams that don't provide `videoHash`.
