## Why

Plexio catalogs can be searched and sorted, but not browsed by genre. Plex already tags every movie and show with genres, and Plexio already returns those tags on each catalog item (`genres` on the meta preview), yet a user can't ask for "Comedy in my Movies library". Stremio and Nuvio both render a genre picker for a catalog when the manifest declares a `genre` extra with options, so the missing piece is entirely on the addon side.

## What Changes

- The manifest declares a `genre` extra on each section catalog, with `options` set to the genres that exist in that Plex library section (fetched from Plex, so they match the user's own library).
- The catalog endpoint accepts a `genre` extra and returns only items in that genre. It combines with the existing `search`, `sort`, and `skip` extras.
- Catalog extra values are URL-decoded, so multi-word genres such as `Science Fiction` round-trip correctly.
- If Plex can't be reached while the manifest is built, the manifest is still returned and that catalog simply omits genre options, rather than failing the whole install.
- No configuration change and no change to the install URL format. Existing installs gain genres the next time the client refetches the manifest.

## Capabilities

### New Capabilities
- `plexio-catalog-genres`: Genre options in the manifest's catalog extras, sourced per Plex library section, and genre filtering of catalog results, including fallback behavior when genres can't be fetched.

### Modified Capabilities

(none; `openspec/specs/` has no existing Plexio specs)

## Impact

- `repos/plexio/plexio/plex/media_server_api.py`: new `get_section_genres` call (`/library/sections/{id}/genre`); `get_section_media` gains a `genre` filter.
- `repos/plexio/plexio/routers/addon.py`: `get_manifest` now makes Plex calls (one per configured section, concurrently, cached) and needs the HTTP client and cache dependencies; `get_catalog` parses and URL-decodes the `genre` extra.
- `repos/plexio/plexio/cache.py` usage: section genre lists are cached with a short TTL.
- Runtime: manifest requests now hit the user's Plex server, so they are slower than the current static manifest and depend on the server being reachable (mitigated by caching and the no-genres fallback).
- Frontend: no change.
- Out of scope: multi-genre filtering, other Plex filters (year, decade, content rating, collection), and a genre catalog that spans sections.
