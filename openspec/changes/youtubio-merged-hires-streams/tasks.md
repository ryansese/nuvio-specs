## 1. Format selection

- [x] 1.1 Add a helper that groups video-only formats by height and picks H.264 first, with codec fallback
- [x] 1.2 Add a helper that picks the best audio-only format, preferring AAC
- [x] 1.3 Extend `parseStream` to emit `Merged <height>p` entries when the flag is on, leaving existing entries untouched

## 2. Mux route

- [x] 2.1 Add `GET /mux` that validates `v` and `a` (https, allowed YouTube media hosts)
- [x] 2.2 Spawn ffmpeg with stream copy and fragmented MP4 output, piped to the response
- [x] 2.3 Kill ffmpeg on client disconnect and cap concurrent mux processes
- [x] 2.4 Detect missing `ffmpeg` and skip merged entries with a logged error

## 3. Configuration

- [x] 3.1 Add the flag to `defaultConfig` (default off) and read it in `parseStream`
- [x] 3.2 Add the setting to the configuration page template

## 4. Packaging and docs

- [x] 4.1 Install `ffmpeg` in the Dockerfile
- [x] 4.2 Document the feature, flag and `ffmpeg` requirement in the README
- [x] 4.3 ~~Bump the version in `package.json` when the change is merged to main (not on the feature branch)~~ Skipped: no new release is planned yet, so the version stays as is

## 5. HLS delivery mode (no server bandwidth)

- [x] 5.1 Extend format selection to pick HLS video-only (`m3u8_native`) and audio-only pairs, H.264 first
- [x] 5.2 Add `GET /merged.m3u8` returning the synthetic master playlist from query parameters; 400 on non-googlevideo URLs
- [x] 5.3 Add the delivery mode setting (`mergedProxy` checkbox, default off = `hls`) to `defaultConfig` and the configuration page; point `Merged` entries at the matching route
- [x] 5.4 Make `hls` mode independent of `ffmpeg`; keep the missing-`ffmpeg` handling for `mux` mode only
- [x] 5.5 Document the same-IP requirement and mode choice in the README

## 6. Verification

- [x] 6.1 Run `yt-dlp -F` on a sample video and confirm video-only 1080p+ and audio-only formats exist
- [x] 6.2 Confirm `Merged` entries appear in the meta and `/stream` responses only when the flag is on
- [x] 6.3 Play a mux URL in mpv or VLC and confirm video and audio
- [x] 6.4 Verify with yt-dlp 2026.08.19 that DASH formats lack `initRange`/`indexRange`, HLS variants are video-only or audio-only, and a hand-built master (312 + 234) probes as h264 1080p + AAC and extracts 5 s at 2:00 with `-c copy` (same IP)
- [x] 6.5 Run the addon and request `/stream` with the flag on, off and in proxy mode; `ffmpeg` on the returned `Merged 1080p` URL extracts 5 s at 2:00 as h264 1920x1080 + AAC (same IP). Not yet tried in mpv or VLC
- [ ] 6.6 Test `hls` mode from a different IP than the addon to confirm the expected 403, and `mux` mode as the workaround
- [ ] 6.7 Test playback in Stremio and the Nuvio TV client
