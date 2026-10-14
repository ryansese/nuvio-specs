## Context

The catalog handler in `repos/YouTubio/addon.js` fetches a page with `-I skip+1:skip+100 --yes-playlist` and maps `videos.entries` (or `[videos]`) through `parseMeta`. The only post-filter is `meta !== null`, which never matches. yt-dlp is run with `-i`, `--ignore-no-formats-error` and `--flat-playlist`, so unavailable videos are not rejected; they arrive as sparse entries with a placeholder title.

`parseMeta` falls back to `Unknown Title` when no title is present, and may replace the title with a DeArrow title, so the raw value has to be checked before that happens.

## Goals / Non-Goals

**Goals:**
- Unavailable videos no longer appear in catalog lists.
- Available entries and their order are unchanged.

**Non-Goals:**
- Changing the meta handler's `available: true` on `videos[]`.
- Matching on the yt-dlp `availability` field.
- Back-filling pages so each one still has 100 items.

## Decisions

- **Key on the raw title.** Treat an entry as unavailable when its raw `title`, trimmed and compared case-insensitively, is `[Deleted video]` or `[Private video]`, or when it is missing or empty. The check runs on yt-dlp's value, before DeArrow and before the `Unknown Title` fallback, so neither can hide the placeholder.
- **Filter in the catalog handler.** Skip unavailable entries before or inside the existing `.filter` after `parseMeta`. Single videos and channel pages that are wrapped as one item go through the same path.
- **Empty result is valid.** If every entry is dropped the handler returns `{ metas: [] }`, the same shape it already returns on error.

## Risks / Trade-offs

- **Short pages** → a page can hold fewer than 100 items after filtering, and `skip` is not adjusted. Accepted: clients keep paging until a page is empty.
- **Legitimately untitled video dropped** → flat-playlist output always supplies titles for real videos, so this is not expected.
- **Placeholder text could change in yt-dlp** → the list is a small constant in one place and is easy to extend.

## Migration Plan

No config or data migration. Deploy the new addon version. Rollback is reverting the version.

## Open Questions

- Should the yt-dlp `availability` field (`private`, `needs_auth`, and similar) be matched as well? Not needed for the placeholder titles seen so far.
