## Context

`parseStream` (`repos/YouTubio/addon.js`) filters `video.formats` to entries with both `acodec` and `vcodec` set, unless `showBrokenLinks` is on. With `showBrokenLinks` the high-resolution video-only formats appear but play without sound, because nothing merges the audio.

Stremio and Nuvio both consume the standard stream object, which carries a single `url`. Video and audio therefore have to be combined into one URL before the client sees it. The addon already proxies media through `/stream/:url` (HLS manifest rewriting for SponsorBlock), so a second proxy route fits the existing architecture.

## Goals / Non-Goals

**Goals:**
- Offer 1080p and higher with audio in both Stremio and Nuvio, using only standard stream fields.
- Leave existing streams and their ordering untouched.
- Make the feature opt-in because it adds server load.

**Non-Goals:**
- Re-encoding or transcoding.
- A Nuvio-only dual-URL stream field (follow-up).
- Perfect seeking in the first version.
- Merging for HLS-only or live streams.

## Decisions

- **Server-side ffmpeg mux over synthetic HLS or client-side merge.** `ffmpeg -i <v> -i <a> -c copy -movflags frag_keyframe+empty_moov -f mp4 pipe:1` works in every client with no client changes. A synthetic HLS master would need DASH files wrapped as byte-range playlists with uneven player support. A client-side merge needs a new stream field that Stremio does not understand.
- **Format pairing.** For each distinct video height, choose the best video-only format and pair it with the best audio-only format. Prefer H.264 (`avc1`) video with AAC (`mp4a`) audio, since they play in MP4 on most TV clients. Fall back to VP9/AV1 with Opus only if no H.264 format exists at that height, in which case the container is WebM/Matroska rather than MP4.
- **Additive stream entries.** Emit `name: "Merged <height>p"` with `behaviorHints.bingeGroup: "Merged <height>p"` and `notWebReady: true`, keeping the existing ordering and fields otherwise.
- **Config flag, default off.** Stored in the encrypted user config, read with `userConfig.x ?? defaultConfig.x` like `showBrokenLinks`.
- **`/mux` route.** Takes `v` and `a` as encoded source URLs, spawns ffmpeg, pipes stdout to the response, and kills ffmpeg when the client disconnects.

## Risks / Trade-offs

- **Bandwidth and CPU on the addon host** → opt-in flag; stream copy only, no transcoding.
- **Open proxy / SSRF via `v` and `a`** → accept only `https` URLs on googlevideo.com hosts (or hosts present in the formats just returned), and cap concurrent mux processes.
- **Seeking** → fragmented MP4 piped from stdout is not seekable; accept for v1, revisit with range support or a temp file.
- **Codec support on TV clients** → prefer H.264/AAC; document that VP9/AV1 fallbacks may fail on some devices.
- **Expiring source URLs** → googlevideo URLs expire; the mux URL is built per request and is not cached beyond the existing TTL.
- **`ffmpeg` missing** → feature disabled with a logged error and no merged streams emitted.

## Migration Plan

Deploy the image with `ffmpeg`. The flag defaults to off, so existing installs are unaffected. Rollback is turning the flag off or reverting the image.

## Open Questions

- Maximum height to offer (cap at 1080p or allow 4K)?
- Whether to add range support for seeking in the first release.
