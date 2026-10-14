## Why

YouTubio's catalog handler maps every yt-dlp entry through `parseMeta`, which never returns null, and yt-dlp runs with `-i`, `--ignore-no-formats-error` and `--flat-playlist`. Deleted and private videos therefore come back as sparse entries (title `[Deleted video]` or `[Private video]`, or no title at all, which `parseMeta` turns into `Unknown Title`) and show up in Stremio and Nuvio lists as items that cannot be played.

## What Changes

- Drop catalog entries whose raw yt-dlp title is `[Deleted video]` or `[Private video]`, or is missing or empty, before they are turned into metas.
- Entries with any other title are returned exactly as before, in the same order.
- Only the catalog response changes. The meta and stream handlers are untouched.

## Capabilities

### New Capabilities
- `youtubio-unavailable-filter`: Detection of unavailable videos from the raw yt-dlp title and their removal from catalog results.

### Modified Capabilities

(none; `openspec/specs/` has no existing YouTubio specs)

## Impact

- `repos/YouTubio/addon.js`: the catalog handler and the `.filter` after `parseMeta`. This is reference only for this workspace; implementation is a separate task in the YouTubio repo.
- Clients: no change. Stremio and Nuvio simply receive fewer items.
- Out of scope: the hardcoded `available: true` in the meta handler's `videos[]`, the stream handler, and matching on the yt-dlp `availability` field.
