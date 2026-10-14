## Why

YouTubio's `parseStream` keeps only yt-dlp formats that contain both video and audio (`acodec != none && vcodec != none`). YouTube serves anything above roughly 720p as separate video-only and audio-only DASH formats, so 1080p and higher never appear in the stream list of Stremio or Nuvio, even though yt-dlp returns them.

## What Changes

- Add extra **"Merged <height>p"** streams to the meta `videos[].streams` and `/stream` responses. Each pairs a video-only format with the best compatible audio-only format.
- Add a `GET /merged.m3u8` route that returns a small synthetic HLS master playlist referencing yt-dlp's video-only and audio-only HLS media playlists. The player fetches the segments directly from googlevideo, so **no media passes through the addon server**. This is the default delivery mode.
- Keep the `GET /mux` route (ffmpeg `-c copy` to fragmented MP4) as an opt-in **proxy mode** for setups where the player cannot reach googlevideo with the addon's resolved URLs (different IP, see Impact).
- Add a per-user config flag, **off by default**, that enables merged streams, plus a delivery mode setting (`hls` default, `mux` proxy).
- Install `ffmpeg` in the Docker image and document it.
- Existing streams (YT-DLP Player, SB Player, Stremio Player, External Player, channel links) are unchanged. Merged streams are additive and use the standard single-`url` stream shape, so Stremio and Nuvio both consume them with no client change.

## Capabilities

### New Capabilities
- `youtubio-merged-streams`: Selection of video-only and audio-only format pairs, the `Merged <height>p` stream entries, the `/mux` route, and the config flag that gates them.

### Modified Capabilities

(none; `openspec/specs/` has no existing YouTubio specs)

## Impact

- `repos/YouTubio/addon.js`: `parseStream`, new `/merged.m3u8` and `/mux` routes, `defaultConfig`, and the config UI template. This is reference only for this workspace; implementation is a separate task in the YouTubio repo.
- `repos/YouTubio/Dockerfile`: add `ffmpeg`.
- Runtime: in `hls` mode the addon serves only a few hundred bytes per play. In `mux` mode it uses server bandwidth and CPU.
- Constraint: googlevideo URLs are bound to the IP that resolved them (`ip/<addr>` in the URL path) and expire. `hls` mode only works when the addon and the player share a public IP (home server, LAN); a hosted addon must use `mux` mode. Verified on yt-dlp 2026.08.19 with a same-IP ffmpeg test; cross-IP behavior is expected to return 403 but is untested.
- Out of scope: a Nuvio-specific dual-URL stream field (possible follow-up; needs client specs).
