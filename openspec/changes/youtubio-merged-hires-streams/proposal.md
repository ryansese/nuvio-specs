## Why

YouTubio's `parseStream` keeps only yt-dlp formats that contain both video and audio (`acodec != none && vcodec != none`). YouTube serves anything above roughly 720p as separate video-only and audio-only DASH formats, so 1080p and higher never appear in the stream list of Stremio or Nuvio, even though yt-dlp returns them.

## What Changes

- Add extra **"Merged <height>p"** streams to the meta `videos[].streams` and `/stream` responses. Each pairs a video-only format with the best compatible audio-only format.
- Add a `GET /mux` route that combines the two source URLs into a single fragmented MP4 response using `ffmpeg -c copy` (no re-encoding).
- Add a per-user config flag, **off by default**, that enables merged streams, because they proxy media through the addon server.
- Install `ffmpeg` in the Docker image and document it.
- Existing streams (YT-DLP Player, SB Player, Stremio Player, External Player, channel links) are unchanged. Merged streams are additive and use the standard single-`url` stream shape, so Stremio and Nuvio both consume them with no client change.

## Capabilities

### New Capabilities
- `youtubio-merged-streams`: Selection of video-only and audio-only format pairs, the `Merged <height>p` stream entries, the `/mux` route, and the config flag that gates them.

### Modified Capabilities

(none; `openspec/specs/` has no existing YouTubio specs)

## Impact

- `repos/YouTubio/addon.js`: `parseStream`, a new `/mux` route, `defaultConfig`, and the config UI template. This is reference only for this workspace; implementation is a separate task in the YouTubio repo.
- `repos/YouTubio/Dockerfile`: add `ffmpeg`.
- Runtime: higher server bandwidth and CPU when merged streams are played.
- Out of scope: a Nuvio-specific dual-URL stream field (possible follow-up; needs client specs).
