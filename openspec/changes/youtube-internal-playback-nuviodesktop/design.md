## Context

NuvioDesktop (`repos/NuvioDesktop`, Kotlin Multiplatform / Compose Desktop) shares the commonMain layout of NuvioMobile: `StreamParser`, `StreamItem`, `StreamDestination.openSelectedStream`, `PlayerLaunch`, and the trailer resolver (`fullCommonMain` sources compiled into `desktopMain` via `build.gradle.kts`).

Desktop differences:
- The internal player is libmpv behind per-OS JNI bridges (`desktopMain/native/{macos,linux,windows}`), driven by `NativePlayerController` (`attach(sourceUrl, sourceHeaders, ...)`, `:116`).
- `PlatformPlayerSurface` receives `sourceAudioUrl`, but `PlayerEngine.desktop.kt` (~`:58`) does not forward it to `NativePlayerSurface`; `attach`/`create` have no audio parameter.
- `AppFeaturePolicy.desktop` sets `externalPlayerSupported = false`, so external launch never applies; Windows also sets `trailerPlaybackMode = EXTERNAL`.
- `useYoutubeChunkedPlayback` is ignored.
- Process spawning is established (`P2pStreamingEngine.desktop.kt` runs TorrServer).

## Goals / Non-Goals

**Goals:**
- Same parse/resolve/play behavior as NuvioMobile for YouTube streams, on macOS, Windows and Linux.
- Audible playback when the resolved source has separate audio.
- No stalling from throttled googlevideo downloads.

**Non-Goals:**
- Enabling the external player on desktop.
- A bundled yt-dlp. It is kept as a documented alternative resolver, not part of this change.
- Changing trailer behavior on Windows.

## Decisions

1. **Share the commonMain changes with NuvioMobile** (`ytId` modeling, parser fix, `openSelectedStream` hook, no persistence of resolved URLs). They are made independently in this repo but follow the same design as `youtube-internal-playback-nuviomobile`.
2. **Forward `sourceAudioUrl` to mpv.** Thread it through `PlayerEngine.desktop.kt` -> `NativePlayerController.attach` -> `NativePlayerBridge.create` -> each native bridge, applying mpv's `audio-file` option. Alternative: ask the extractor for a muxed/HLS source to avoid native work; chosen as the fallback if a bridge cannot be changed, since muxed formats cap quality.
3. **Throttling mitigation** through mpv options (`stream-lavf-o`, cache/demuxer settings) or preferring HLS; decide after measuring with a long video. There is no desktop equivalent of `YoutubeChunkedDataSourceFactory`.
4. **Ignore external-player logic** for YouTube; the internal path is unconditional. On Windows, the trailer EXTERNAL mode must not affect stream playback.
5. **Process-based resolver kept as future option** (modeled on `P2pStreamingEngine.desktop.kt`) if the in-app extractor proves unreliable.

## Risks / Trade-offs

- [Native bridge changes on three platforms] -> muxed/HLS fallback; verify each OS.
- [Throttled playback] -> measure; adjust mpv options or prefer HLS.
- [Extractor fragility] -> surface an error message.
- [Drift between Mobile and Desktop commonMain] -> keep shared logic in one helper and mirror tests.

## Open Questions

- Do the existing native bridges already accept extra mpv options that would allow `audio-file` without a JNI signature change?
- Are muxed formats acceptable in quality if audio forwarding is deferred?
