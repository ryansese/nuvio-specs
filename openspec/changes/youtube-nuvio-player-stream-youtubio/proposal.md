## Why

YouTubio returns `Stremio Player` (a `ytId` stream) for Stremio users but has no item for Nuvio users. Nuvio clients are gaining the ability to play a YouTube video in their own built-in player when the user taps a specifically named stream, so YouTubio needs to expose that stream.

## What Changes

- Add one new stream named `Nuvio Player` for `youtube.com` videos, with the YouTube watch URL as `externalUrl` (no `url`) and the description `Click to watch using Nuvio's built-in Youtube Player`.
- Keep every existing stream, name, field, deep link, manifest entry and install button exactly as it is. Nothing is renamed or removed.
- Emit the stream under the same conditions as `Stremio Player` (a `youtube.com` video id, or a live channel id), directly after it.
- The name `Nuvio Player` is a contract with the Nuvio clients, which recognise the stream by that exact name together with its YouTube `externalUrl`.

## Capabilities

### New Capabilities
- `youtube-nuvio-player-stream-youtubio`: the `Nuvio Player` stream YouTubio returns for YouTube videos.

### Modified Capabilities

## Impact

- `YouTubio/addon.js`: `parseStream` only. The file uses CRLF line endings, which must be preserved.
- Consumers: `youtube-internal-playback-nuviotv`, `youtube-internal-playback-nuviomobile` and `youtube-internal-playback-nuviodesktop` play this stream in-app. Stremio and other clients ignore it or open the watch URL.
