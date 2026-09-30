---
paths:
  - "repos/YouTubio/**"
---

# YouTubio (repos/YouTubio)

This file provides guidance to Claude Code (claude.ai/code) when working with code in the YouTubio submodule.

## What this is

YouTubio is a Stremio addon (Express server) that exposes YouTube (and any other yt-dlp-supported site) as Stremio catalogs, metas, streams and subtitles. Third-party upstream (`xXCrash2BomberXx/YouTubio`); it is a git submodule of the `nuvio-specs` workspace, so git operations here affect this repo, not the parent.

The whole server is a single file, `addon.js` (~1450 lines, CommonJS). There is no test suite, linter, or build step.

## Commands

- `npm install` — install deps (`package-lock.json` is gitignored)
- `npm start` — runs `node --env-file-if-exists=.env addon.js` (default port 7000, override with `PORT`)
- `npm run local` — start plus an `ngrok` tunnel on 7000
- Requires the `yt-dlp` binary on `PATH` (the Dockerfile installs it via pip with `curl-cffi`), and Node >= 22 for `--env-file-if-exists`.
- Set `DEV_LOGGING=1` to print errors and the generated encryption key, and to point logo URLs at `main` instead of the `v${VERSION}` tag.

## Architecture

**Config is carried in the URL.** Each user's settings (catalogs, cookies, Gemini key, flags) are JSON, encrypted with AES-256-GCM (`encrypt`/`decrypt`, key from `ENCRYPTION_KEY` base64, otherwise a random key generated at startup, which invalidates all existing install links on restart). The encrypted blob is the `/:config/` path segment of every Stremio route. `POST /encrypt` produces it; `decryptConfig` reads it and merges over `defaultConfig`. Cookies live under `userConfig.encrypted.auth`.

**yt-dlp is the data source.** `runYtDlpWithAuth(url, encryptedConfig, args)` is the single choke point: it writes the user's cookies to a temp file, runs yt-dlp with `-J --flat-playlist`, parses the JSON, and deletes the file. Results are cached in `node-cache` (TTL env var, default 3600s) only for URLs matching the channel/playlist/video ID regexes, and never when `markWatchedOnLoad` is on.

**Stremio routes** (all under `/:config/`): `manifest.json` (builds catalogs from user config), `catalog/:type/:id/:extra?.json`, `meta/:type/:id.json`, `stream/:type/:id.json`, `subtitles/:type/:id.json`, plus `/:config/playlists` (lists the user's YouTube playlists for the config UI). `parseMeta` and `parseStream` convert yt-dlp entries into Stremio objects; IDs are prefixed `yt_id:` (`prefix`), with a `Reversed` variant for reverse-ordered playlists. `toYouTubeURL` maps a Stremio catalog id plus search/sort extras back to a YouTube URL (see the search-catalog `{term}`/`{sort}` convention in the README).

**Stream post-processing.** `/stream/:url` (unprefixed by config) fetches an HLS manifest and `cutM3U8` removes SponsorBlock segment ranges. Segments come from `getSponsorBlockSegments`, with an optional Gemini fallback (`getGeminiSegments`). DeArrow supplies titles and thumbnails (`runDeArrow`, `getDeArrowThumbnail`).

**Config UI.** `GET /` and `/:config?/configure` return a large inline HTML/JS template string (roughly lines 893–1440) — edits to the configure page are edits inside that template literal.

## Environment variables

`PORT`, `TTL`, `ENCRYPTION_KEY`, `DEV_LOGGING`, `YTDLP_EXTRACTORS` (default `all`), `NO_DEARROW`, `NO_SPONSORBLOCK` (disable the feature and comment out its UI), `EMBED` / `YTDLP_EXTRACTORS_EMBED` (extra HTML injected into the config page), `SPACE_HOST` (public host shown in the startup log).

## Releases

CI (`.github/workflows/check-ver.yml`) fails a push to `main` unless `package.json` `version` is greater than the latest GitHub release. `release-please.yaml` handles release automation. Bump the version in `package.json` with any user-facing change (recent history: "Bump version from X to Y").
