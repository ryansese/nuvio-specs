## 1. Stream

- [x] 1.1 Add the `Nuvio Player` stream (`externalUrl` = `video.webpage_url`, description `Click to watch using Nuvio's built-in Youtube Player`, `behaviorHints.filename`, no `url`) to `parseStream` in `addon.js`, under the same condition as `Stremio Player` and directly after it
- [x] 1.2 Confirm no existing stream, name, field, meta link, manifest entry or configure-page element changed

## 2. Verification

- [x] 2.1 Run the addon and call the stream route for a `youtube.com` video: exactly one new `Nuvio Player` stream appears, and a non-YouTube video has none
- [x] 2.2 Confirm `git diff --stat` for `addon.js` shows no whole-file diff (CRLF preserved)
