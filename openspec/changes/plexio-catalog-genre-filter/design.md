## Context

Plexio's manifest (`routers/addon.py:get_manifest`) is built only from the decoded config and makes no Plex calls. Each configured section becomes a catalog with `skip`, `search`, and `sort` extras. `get_catalog` parses the extra path segment with `dict(e.split('=') for e in extra.split('&'))` (no URL-decoding) and calls `get_section_media`, which queries `GET /library/sections/{id}/all` with `title`, `sort`, and `X-Plex-Container-Start/Size=100`.

In the Stremio protocol, a catalog's selectable extra options must be listed in the manifest. Clients (Stremio, Nuvio) render them as a picker and send the chosen value back as `genre=<value>` in the catalog URL. So genre support needs (a) the genre list at manifest time and (b) a filter at catalog time.

Plex exposes per-section genres at `GET /library/sections/{id}/genre`, which returns `MediaContainer.Directory[]` entries with `key` (a numeric tag id) and `title`. `/library/sections/{id}/all` accepts `genre=<id>` as a filter.

## Goals / Non-Goals

**Goals:**
- Per-section genre options in the manifest that match the user's library.
- Genre filtering that composes with the existing search, sort, and paging.
- A manifest that never fails because of a genre lookup.

**Non-Goals:**
- Multi-select genres, or other Plex filters (year, decade, content rating, collection, unwatched).
- Changes to the config UI or the install URL.
- Translating or normalizing genre names beyond what Plex returns.

## Decisions

**1. Fetch genres live at manifest time, cached.** `get_manifest` takes the HTTP client and cache dependencies and fetches `/library/sections/{id}/genre` for every configured section concurrently (`asyncio.gather(..., return_exceptions=True)`).
- *Alternative: store genres in the config at install time (frontend fetches them).* No manifest-time Plex call, but genres go stale until reinstall, the config URL grows, and the frontend has to change. Rejected.
- *Alternative: static list of common genres.* Doesn't match Plex agents' genre names and offers genres the library lacks. Rejected.

**2. Clients send genre titles; the server resolves the title to a Plex genre id.** Manifest options are titles (what users see). In `get_catalog`, the title is matched case-insensitively against the cached section genre list to get its `key`, which is passed to Plex as `genre=<key>`. If no match is found, return an empty catalog without querying `/all`.
- *Alternative: pass the title straight to Plex (`genre=Comedy`).* Plex filter behavior with titles is undocumented and inconsistent across versions. Rejected unless verification (task 1.1) shows titles work reliably, in which case the lookup can be dropped.
- *Alternative: encode `id:title` in the options.* Leaks ids into the UI. Rejected.

**3. Cache key and TTL.** Key on the server and section, for example `genres:<discovery_url host>:<section_id>`. Don't use the access token, so shared users of the same server share entries, and the token doesn't end up in Redis keys. Store a JSON list of `{key, title}`. Use a 6-hour TTL, overriding the cache's 24h default, so newly tagged genres appear within the same day. Only successful results are cached. `cache.py` needs a per-call TTL if it doesn't already support one.

**4. Fail open in the manifest, fail normally in the catalog.** In the manifest, any exception from the genre lookup (the `HTTPException` 502/504 from `get_json`, `PlexUnauthorizedError`, a parse error) is swallowed per section and the `genre` extra is omitted. In the catalog, the genre lookup uses the same cached helper; failures propagate like any other catalog Plex error.

**5. URL-decode extras.** Replace the ad-hoc split with `urllib.parse.parse_qsl(extra, keep_blank_values=True)`, which also tolerates `=` in values. Stremio clients percent-encode extra values, so today `Science%20Fiction` would reach Plex encoded.

## Risks / Trade-offs

- [Manifest latency rises from instant to one Plex round-trip, and clients may time out on slow servers] → Concurrent per-section requests, cache hits on repeat fetches, and the existing `PLEX_REQUESTS_TIMEOUT`. Consider a tighter timeout just for the manifest genre calls.
- [Clients cache the manifest, so new genres appear only after a manifest refresh, which can mean a reinstall] → Acceptable. Document it in the README.
- [Shared-server users see a section's genres through the cache entry populated by another user] → The genre list is a property of the section, not of the user, and every user with the config can already list it.
- [A section with many genres yields a long picker] → Plex genre lists are typically under 50 entries; no truncation.
- [Genre title collisions differing only by case] → Take the first match in Plex order.

## Migration Plan

Additive. Deploy the new version; existing installs keep working with the old manifest until the client refetches it, and then they gain the `genre` extra. Rollback is a redeploy of the previous version. Clients that still hold the new manifest then send `genre=` extras the old server ignores, so they get unfiltered results, which is harmless.

## Open Questions

- Does `/library/sections/{id}/all?genre=<title>` work on current Plex Media Server versions? If so, drop the title→id resolution (task 1.1 decides).
- Should the manifest's genre lookups use a shorter timeout than catalog requests?
