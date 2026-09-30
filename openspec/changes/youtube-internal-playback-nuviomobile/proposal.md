## Why

NuvioMobile drops `ytId`-only streams and opens YouTube `externalUrl` streams in the browser. YouTubio now returns a stream named `Nuvio Player` (`youtube-nuvio-player-stream-youtubio`), and NuvioMobile should play it in its built-in player.

## What Changes

- Play a stream named exactly `Nuvio Player` whose `externalUrl` is a YouTube video URL (and that has no `url`) in the built-in player when the user selects it, resolving the video on-device instead of opening a browser or another app.
- Leave every other card unchanged: `External Player` and any other YouTube `externalUrl` card still open the browser, `ytId` handling, autoplay and in-player source switching are unaffected, and no tab or group is added.
- Prefer 1080p and fall back to lower resolutions for this playback (not for trailers), ignore the internal/external/ask preference for it, and never persist the resolved URL.
- Keep the previous behavior where in-app YouTube playback is disabled by the build variant.

## Capabilities

### New Capabilities
- `youtube-stream-playback-nuviomobile`: how NuvioMobile recognises and plays the `Nuvio Player` YouTube stream in its built-in player, including the resolution preference and build-variant gating.

### Modified Capabilities

## Impact

- See `design.md` and `tasks.md` for the affected files in `NuvioMobile`.
- Depends on `youtube-nuvio-player-stream-youtubio` for the stream. Sibling changes: the other two `youtube-internal-playback-*` changes.
