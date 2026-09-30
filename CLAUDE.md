# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

An [OpenSpec](https://github.com/Fission-AI/OpenSpec) workspace (schema: `spec-driven`) for the Nuvio ecosystem. It contains no application code of its own: the source repos are git submodules under `repos/`, and the specs about them live in `openspec/`. Clone with `git clone --recurse-submodules`.

- `openspec/specs/` — main (current-truth) specs; empty so far
- `openspec/changes/` — in-flight changes; `changes/archive/` holds completed ones
- `openspec/config.yaml` — schema and project context
- `.claude/skills/openspec-*` and `.claude/commands/opsx/` — the workflow: `/opsx:explore`, `/opsx:propose`, `/opsx:apply`, `/opsx:sync`, `/opsx:archive`

## Submodules (`repos/`)

The submodules have different upstreams and ownership. Treat them as read-only reference for writing specs unless asked to change code.

| Submodule | Stack | Notes |
|---|---|---|
| `NuvioTV` | Android TV, Kotlin/Gradle (`app/`, `libmpv-android`, `DV7`, `ffmpeg-decoder-downmix`) | Upstream NuvioMedia |
| `NuvioMobile` | Kotlin Multiplatform / Compose (`composeApp`, `androidApp`, `iosApp`) | Upstream NuvioMedia |
| `NuvioDesktop` | Kotlin Multiplatform / Compose, same layout as NuvioMobile plus `desktopSentry` | Upstream NuvioMedia |
| `plexio` | Python addon (`plexio/`, `pyproject.toml`) plus `frontend/`, Docker/fly.io deploy | Stremio addon bridging Plex; the user's own fork (`ryansese/plexio`) |
| `YouTubio` | Node.js single-file addon (`addon.js`) | Stremio YouTube addon; third-party upstream |

Nuvio clients (TV, Mobile, Desktop) are Gradle builds; use each repo's own `./gradlew` (e.g. `./gradlew :app:assembleFullDebug` in NuvioTV). See each repo's README and CONTRIBUTING.md for build details. Plexio and YouTubio are Stremio addons, which are a separate addon ecosystem from the Nuvio clients.

## Working notes

- Run submodule commands from inside the submodule directory; git operations there affect the submodule's own repo, not this one. Bumping a submodule pointer is a commit in this repo.
- There is no build, lint, or test command at the workspace root.
