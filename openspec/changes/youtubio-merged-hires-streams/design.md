## Context

`parseStream` (`repos/YouTubio/addon.js`) filters `video.formats` to entries with both `acodec` and `vcodec` set, unless `showBrokenLinks` is on. With `showBrokenLinks` the high-resolution video-only formats appear but play without sound, because nothing merges the audio.

Stremio and Nuvio both consume the standard stream object, which carries a single `url`. Video and audio therefore have to be combined into one URL before the client sees it. Nuvio's `Stream` model has no second-URL field, and its main player has no external-audio merge (only the trailer player does), so a Nuvio-specific field is not an option without client changes.

yt-dlp (2026.08.19) output for a sample video shows:
- DASH video-only (`299` 1080p avc1) and audio-only (`140` m4a) formats are plain URLs with `http_chunk_size`; they carry no `initRange`, `indexRange` or `fragments`, so a synthetic DASH MPD would require probing `sidx` ranges.
- HLS formats are video-only (`312` 1080p avc1, `617` 1080p vp09, `623` 1440p, `628` 2160p) or audio-only (`233`, `234`). None is muxed.
- yt-dlp leaves `acodec` and `abr` unset on HLS audio-only formats (`audio_ext` is `mp4`, `format_note` "Default, high"/"Default, low"), so they are recognised by `vcodec: none` plus `audio_ext`, and the audio codec is assumed to be `mp4a.40.2`.
- Each HLS URL is a flat VOD media playlist (`EXTINF` plus absolute `googlevideo.com/videoplayback/...` segment URLs, no byte ranges, no `EXT-X-MAP`).
- Segment URLs embed `ip/<resolver ip>` and `expire`. The addon already proxies media through `/stream/:url` (HLS manifest rewriting for SponsorBlock), so a second proxy route fits the existing architecture.

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

- **Synthetic HLS master as the default delivery.** `GET /merged.m3u8` returns a master playlist with the audio playlist as `EXT-X-MEDIA:TYPE=AUDIO` and the video playlist as `EXT-X-STREAM-INF` (with `AUDIO` group, `RESOLUTION`, `BANDWIDTH`, `CODECS`). Both URIs are yt-dlp's own HLS media playlists, so no DASH wrapping is needed and no media flows through the addon. Tested with ffmpeg: both tracks detected (h264 1920x1080 + AAC), seek to 2:00 and 5 s `-c copy` extraction produced synced video and audio.
- **Server-side ffmpeg mux as opt-in proxy mode.** `ffmpeg -i <v> -i <a> -c copy -movflags frag_keyframe+empty_moov -f mp4 pipe:1` works when the player cannot use the resolved URLs directly (different IP). It costs server bandwidth and CPU, so it is only used when the mode is set to `mux`.
- **Rejected: synthetic DASH.** yt-dlp exposes no `initRange`/`indexRange`. **Rejected: muxed HLS passthrough.** yt-dlp returns no HLS variant with audio. **Rejected: client-side merge.** Needs a new stream field Stremio does not understand and Nuvio client changes.
- **Format pairing.** For each distinct video height, choose the best video-only format and pair it with the best audio-only format. Prefer H.264 (`avc1`) video with AAC (`mp4a`) audio. In `hls` mode pair HLS formats (`protocol` `m3u8_native`, e.g. `312` + `234`); in `mux` mode pair DASH https formats. Fall back to VP9 only if no H.264 format exists at that height (HLS VP9 support varies on TV clients); AV1 and Opus are not used in `hls` mode. Heights above 1080p exist only as VP9 in HLS.
- **Additive stream entries.** Emit `name: "Merged <height>p"` with `behaviorHints.bingeGroup: "Merged <height>p"` and `notWebReady: true`, keeping the existing ordering and fields otherwise.
- **Config flag, default off, plus mode.** Stored in the encrypted user config, read with `userConfig.x ?? defaultConfig.x` like `showBrokenLinks`. A second checkbox, `mergedProxy` (default off), selects `mux` (on) over `hls` (off).
- **`/mux` route.** Takes `v` and `a` as encoded source URLs, spawns ffmpeg, pipes stdout to the response, and kills ffmpeg when the client disconnects.

## Risks / Trade-offs

- **IP-bound URLs** → segment URLs carry the resolver's IP and an expiry. `hls` mode works only if the player shares the addon's public IP; otherwise segments likely 403. Document this and let users switch to `mux`. Not tested cross-IP.
- **Stale playlist URLs** → `/merged.m3u8` is stateless: the stream entry carries the video and audio playlist URLs (plus resolution, bandwidth and codecs) as query parameters, validated as https googlevideo.com, and is built per `/stream` request like `/mux`, so it expires with the source URLs.
- **Player strictness** → ffmpeg is lenient; ExoPlayer and libmpv may be stricter about a master with a separate audio group and bare media playlists. Verify in Nuvio TV and Stremio before release.
- **Bandwidth and CPU on the addon host (`mux` mode only)** → opt-in; stream copy only, no transcoding.
- **Open proxy / SSRF via `v` and `a`** → accept only `https` URLs on googlevideo.com hosts (or hosts present in the formats just returned), and cap concurrent mux processes.
- **Seeking** → fragmented MP4 piped from stdout is not seekable; accept for v1, revisit with range support or a temp file.
- **Codec support on TV clients** → prefer H.264/AAC; document that VP9/AV1 fallbacks may fail on some devices.
- **Expiring source URLs** → googlevideo URLs expire; the mux URL is built per request and is not cached beyond the existing TTL.
- **`ffmpeg` missing** → feature disabled with a logged error and no merged streams emitted.

## Migration Plan

Deploy the image with `ffmpeg` (needed for `mux` mode only). The flag defaults to off, so existing installs are unaffected. Existing users who enabled merged streams on the earlier `/mux`-only release keep working: the mode setting defaults to `hls`, so they should switch to `mux` if they host the addon off-network. Rollback is turning the flag off or reverting the image.

## Open Questions

- Maximum height to offer (cap at 1080p or allow 4K)?
- Whether to add range support for seeking in `mux` mode (HLS mode seeks natively).
- Whether to auto-select the mode (e.g. compare the request's client IP with the resolver IP) instead of a manual setting.
