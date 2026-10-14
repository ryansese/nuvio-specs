## 1. Detection

- [x] 1.1 Add a helper that returns true when a raw yt-dlp entry title is `[Deleted video]`, `[Private video]`, missing or empty (trimmed, case-insensitive)

## 2. Catalog filter

- [x] 2.1 Skip unavailable entries in the catalog handler before `parseMeta` applies DeArrow or the `Unknown Title` fallback
- [x] 2.2 Return `{ metas: [] }` when every entry is dropped

## 3. Verification

- [x] 3.1 Run the addon against a playlist that contains a deleted and a private video and confirm both are absent from the catalog response
- [x] 3.2 Confirm the remaining entries are unchanged and in the same order
- [x] 3.3 Confirm channel and single-video catalogs still return their one item

## 4. Release

- [x] 4.1 ~~Bump the version in `package.json` when the change is merged to main (not on the feature branch)~~ Skipped: no new release is planned yet, so the version stays as is
