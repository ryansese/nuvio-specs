## Context

NuvioTV's stream flow:

- **`data/repository/StreamRepositoryImpl.kt`**
  - `fetchStreamsFromAllSources` (lines ~157–300) launches one coroutine per add-on and passes results through a channel as they arrive.
  - If an add-on's response is empty, it tries `fetchInlineStreamsFromMeta` (~616–670), which uses the content ID with trailing numeric segments removed.
  - `dedupKey` (~532) removes duplicates. Plugins and debrid streams merge into the same flow, but they're out of scope here.
- **`core/streams/PlaybackAvailability.kt`**, `supportsStreamResource`: decides which add-ons are queried.
- **`core/player/StreamAutoPlaySelector.kt`**, `orderAddonStreams`: direct-debrid groups first, then add-ons in installed order, then plugins. On Switch only the add-on part applies until 0018.
- **`domain/model/Stream.kt`**: `getStreamUrl`, `isTorrent`, `isExternal`, `isYouTube`, `getDisplayName`, `getDisplayDescription`.
- **`data/mapper/StreamMapper.kt`**: maps the DTO and removes the `Range` header from `proxyHeaders.request`.
- **`ui/screens/stream/StreamScreen.kt`**: left 40% shows the logo, title and episode; right 60% shows the add-on chips and a list of `StreamCard`s.

## Goals / Non-Goals

**Goals:**
- Streams appear progressively, in a stable and predictable order (the user's add-on order).
- Torrent-only and unsupported streams are flagged clearly instead of failing silently in the player.
- A self-contained `PlaybackRequest` struct that 0008 consumes, and that 0010 (autoplay) and 0018 (debrid) can build too.

**Non-Goals:**
- Auto-play, binge groups and remembering the last stream (0010).
- Debrid resolution, plugin scrapers and YouTube.
- Sorting by quality. NuvioTV doesn't sort add-on streams either.

## Decisions

**1. `StreamRepository::fetchAll(type, videoId, onGroup, onDone)`.**
- It starts one HTTP task per selected add-on on 0002's pool. Each task parses the streams, falls back to inline streams when empty, removes duplicates within the group, and posts `onGroup(addonIndex, group)` through `brls::sync`.
- `onDone` fires when all tasks finish, carrying the list of failed add-on names.
- A `generation` token makes Retry or leaving the screen ignore late callbacks.
- Alternative: wait for all responses and render once. Rejected; one slow add-on would block the whole list, which is the main pain point the progressive approach solves.

**2. Pure helpers for unit tests** (`src/streams/stream_rules.{hpp,cpp}`):
- `supportsStreamResource`, `contentIdFromVideoId`, `dedupKey`, `classify(stream)` → {Playable, NeedsDebrid, Unsupported}, `requestHeaders(stream)` (removes `Range`), and `formatSize(bytes)`;
- one URL-encoding helper shared with catalog and meta URLs (Java `URLEncoder` semantics).
- If 0004 or 0006 already add a shared encoder, reuse it rather than adding a second one.

**3. UI.**
- `StreamActivity`, pushed from `DetailsActivity`.
- The chip row is a horizontal `Box` of buttons, rebuilt when a new group arrives (adding a chip never moves focus).
- The stream list is a `RecyclerFrame` with section headers per add-on when "All" is selected. Groups are inserted in add-on order, and focus stays on the focused card when a group is inserted above it.
- Non-playable cards use the disabled text colour and a small label.

**4. The `PlaybackRequest` struct**, defined in `src/streams/playback_request.hpp`, holds:
- url, headers, subtitles[], bingeGroup, filename, videoHash, videoSize, addonName;
- content info: type, contentId, videoId, season, episode, title, episodeTitle, poster, backdrop, logo.

This mirrors the player's navigation arguments in NuvioTV (`ui/screens/player/PlayerNavigationArgs.kt`), minus the torrent and debrid fields.

## Risks / Trade-offs

- [Some add-ons (Torrentio-style) return mostly torrent-only streams, so the picker can look full of unplayable entries] → Flag them clearly, and make "Hide unplayable streams" a 0011 setting. Debrid (0018) makes most of them playable.
- [Inserting groups while the user scrolls can move content] → Insert above the focused group only by adjusting the scroll offset, so the focused card stays put.
- [Large responses (hundreds of streams) are slow to parse on Switch] → Parse on the worker thread, and render through `RecyclerFrame` so only visible cells exist.
