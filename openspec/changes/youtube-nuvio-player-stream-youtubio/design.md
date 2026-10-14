## Context

`parseStream` in `repos/YouTubio/addon.js` emits, for a `youtube.com` video or live channel id, `Stremio Player` (`ytId`) followed by `External Player` (`externalUrl` = watch URL), then channel entries. Nuvio clients cannot use `ytId` on Mobile, and treat a YouTube `externalUrl` as an external (browser) link.

## Goals / Non-Goals

**Goals:**
- Give Nuvio users an entry that plays the video in Nuvio's built-in player.

**Non-Goals:**
- Renaming, removing or relabeling any existing item, or changing the manifest, meta links, deep links or configure page.
- Client detection on the server.
- Client changes (see the `youtube-internal-playback-*` changes).

## Decisions

1. **Add, don't rename.** A single new `Nuvio Player` stream is added and all existing output is preserved, so Stremio users and existing clients are unaffected.
2. **Link is the plain watch URL.** `externalUrl` = `video.webpage_url`, the same value `External Player` uses. Nuvio has no deep link that plays a video (its `nuvio://` links open details pages), and the clients resolve the YouTube URL on-device.
3. **The name is the marker.** Clients recognise the stream by the exact name `Nuvio Player` plus a YouTube `externalUrl` and no `url`. Other clients, and the same URL under a different name, keep their current behavior.
4. **Description.** `Click to watch using Nuvio's built-in Youtube Player`, mirroring the wording of `Stremio Player`.
5. **Placed right after `Stremio Player`** under the same emission condition, keeping the two player options adjacent.
6. **Preserve CRLF.** `addon.js` uses CRLF; edit with a tool or script that keeps it.

## Risks / Trade-offs

- Renaming the stream later breaks the clients' recognition. The name is documented here and in each client spec.
- A client without this support shows `Nuvio Player` as an ordinary external card that opens the YouTube watch page.
- One extra list item for Stremio users, who cannot use it.
