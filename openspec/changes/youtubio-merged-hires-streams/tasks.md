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
- [x] 4.3 Bump the version in `package.json`

## 5. Verification

- [x] 5.1 Run `yt-dlp -F` on a sample video and confirm video-only 1080p+ and audio-only formats exist
- [x] 5.2 Confirm `Merged` entries appear in the meta and `/stream` responses only when the flag is on
- [x] 5.3 Play a mux URL in mpv or VLC and confirm video and audio
- [ ] 5.4 Test playback in Stremio and the Nuvio TV client
