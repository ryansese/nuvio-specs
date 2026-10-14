## 1. Verify Plex API behavior

- [ ] 1.1 Against a real Plex server, confirm the shape of `GET /library/sections/{id}/genre` (`Directory[].key`, `.title`) for a movie and a show section, and whether `/all?genre=<id>` and `/all?genre=<title>` both filter correctly. Record the result in design.md's Open Questions.

## 2. Plex API layer

- [x] 2.1 Add `get_section_genres(client, url, token, cache, section_id)` in `plex/media_server_api.py`, returning `[{key, title}]` and reading/writing the cache under `genres:<host>:<section_id>` with a 6h TTL. Never cache failures.
- [x] 2.2 Add per-call TTL support to `cache.py` (memory and Redis backends) if it doesn't already exist.
- [x] 2.3 Add an optional `genre` parameter (Plex genre id) to `get_section_media` and pass it as the `genre` query param.

## 3. Manifest

- [x] 3.1 Inject the HTTP client and cache into `get_manifest`; when a configuration is present, fetch genres for all sections concurrently with `asyncio.gather(..., return_exceptions=True)`.
- [x] 3.2 Append `{'name': 'genre', 'isRequired': False, 'options': [titles]}` to each catalog's extras when its lookup succeeded and returned at least one genre; otherwise omit it.
- [x] 3.3 Confirm the unconfigured manifest path makes no Plex calls.

## 4. Catalog

- [x] 4.1 Replace the extras parsing in `get_catalog` with `urllib.parse.parse_qsl` so values are URL-decoded.
- [x] 4.2 When a `genre` extra is present, resolve it case-insensitively to a Plex genre id through `get_section_genres`; return `StremioCatalog(metas=[])` if there's no match, otherwise pass the id to `get_section_media`.

## 5. Verify and release

- [x] 5.1 `ruff check .` and `ruff format --check .` pass on the files this change touches in `repos/plexio` (the repo already had a C901 error in `models/plex.py` and a format issue in `models/utils.py`).
- [ ] 5.2 With `docker-compose up`, check every scenario in `specs/plexio-catalog-genres/spec.md` by curl: genre options in the manifest, filtering, combination with sort/search/skip, an unknown genre, `Science%20Fiction`, and the manifest with Plex unreachable (wrong discovery URL in the config).
- [ ] 5.3 Install the addon in Stremio or Nuvio and confirm the genre picker appears and filters a catalog.
- [x] 5.4 Note in the README that genre lists update when the client refreshes the manifest.
- [ ] 5.5 Bump the version in `plexio/__init__.py` and `frontend/package.json`, then bump the `repos/plexio` submodule pointer in this workspace.
