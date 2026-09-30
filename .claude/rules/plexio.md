---
paths:
  - "repos/plexio/**"
---

# plexio (repos/plexio)

This file provides guidance to Claude Code (claude.ai/code) when working with code in the plexio submodule.

## What this is

Plexio is a Stremio addon that exposes a Plex library to Stremio. It is a FastAPI backend (`plexio/`) plus a React/Vite configuration UI (`frontend/`), shipped as one Docker image. This checkout is a git submodule of the `nuvio-specs` workspace (the user's fork, `ryansese/plexio`); git operations here affect this repo, not the parent.

## Commands

There are no tests in this repo. Linting only:

- Backend: `pip install -e '.[dev]'`, then `ruff check .` and `ruff format .` (config in `pyproject.toml`: line length 88, single quotes, isort rules).
- Frontend (from `frontend/`): `npm install`, `npm run dev`, `npm run build` (runs `tsc` first), `npm run lint` (zero warnings allowed). Prettier config is in `frontend/package.json` and sorts imports.
- Full local stack: create a `.env`, then `docker-compose up --build`. The nginx proxy serves everything on `:80`, routing `/api/*` and `*.json` to the backend (`:8000`, uvicorn `--reload`) and everything else to the Vite dev server (`:5173`). Redis is on host port `6399`.

## Architecture

**Stateless, config-in-URL.** The addon holds no per-user state. A user's configuration (`AddonConfiguration` in `plexio/models/addon.py`: Plex access token, discovery/streaming URLs, selected library sections, transcode options) is JSON, camelCase-aliased, base64-encoded, and embedded in the addon install URL: `/{installation_id}/{base64_cfg}/...`. `dependencies.get_addon_configuration` decodes it on every request. The frontend builds that URL; the backend never stores the config.

**Backend request flow.**
- `main.py` builds the FastAPI app. The `lifespan` creates one shared `aiohttp.ClientSession` and one cache, passed via lifespan state and read through `dependencies.get_http_client` / `get_cache`.
- `routers/addon.py` implements the Stremio addon protocol: `manifest.json`, `catalog`, `meta`, `stream`. The manifest builds one catalog per configured Plex section. Without a config it returns a `configurationRequired` manifest.
- `routers/configuration.py` (`/api/v1/test-connection`) lets the frontend probe a Plex server through the backend.
- `plex/media_server_api.py` holds all Plex Media Server calls (`get_section_media`, `get_media`, `get_all_episodes`, `stremio_to_plex_id`, and the `SORT_OPTIONS` map). `plex/utils.py:get_json` is the single HTTP helper. It maps Plex failures to 502/504 `HTTPException`s, which `main.py`'s Sentry `before_send` deliberately drops.
- `models/plex.py` parses Plex responses and converts them to Stremio types (`to_stremio_meta`, `get_stremio_streams` for direct and transcoded streams). `models/stremio.py` holds the Stremio response models. `models/utils.py` has the `plexio:` id <-> Plex GUID conversion plus language emoji tables.

**ID handling.** Stremio requests arrive as IMDB ids (`tt...`), as Plexio's own `plexio:<guid>` ids (media with no IMDB match), or as raw Plex ids. `get_stream` resolves `tt` ids to Plex GUIDs through `stremio_to_plex_id`, which caches results (`cache.py`, memory or Redis, 24h TTL). Shared (non-owned) Plex servers can't do this lookup with the user's token, so `PLEX_MATCHING_TOKEN` supplies an owner token.

**Config:** `plexio/settings.py` (`pydantic-settings`, env vars `CORS_ORIGIN_REGEX`, `PLEX_REQUESTS_TIMEOUT`, `CACHE_TYPE`, `REDIS_URL`, `PLEX_MATCHING_TOKEN`; `SENTRY_DSN` is read by `sentry_sdk.init()`). The README documents the defaults.

**Frontend.** `frontend/src` is React 18 + Tailwind + shadcn/Radix UI (`components/ui`), react-hook-form + zod. It handles Plex OAuth login, then lists servers and library sections (`hooks/`, `services/PlexService.tsx`, `PMSService.tsx`) and generates the install URL. `BackendService.tsx` calls `/api/v1/test-connection` on `window.location.origin`. The `@` alias maps to `src`.

**Production image.** The `Dockerfile` is multi-stage: it builds the frontend, then copies `dist` into an NGINX Unit image that runs the Python app as an ASGI app (`unit-nginx-config.json`). Unit routes `/api/*` and `*json` to the backend, and serves the SPA from `/app/frontend` for everything else. `local.Dockerfile` is backend-only for dev.

## Versioning and release

The version lives in two places that must stay in sync: `plexio/__init__.py` (`__version__`, also reported in the Stremio manifest) and `frontend/package.json`. `fly.toml` pins the published image tag (`ghcr.io/vanchaxy/plexio:<version>`) for fly.io deploys. The GitHub Action builds and pushes the image to GHCR when a release is published.
