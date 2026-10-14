## Purpose

TBD: Define how the YouTubio catalog removes deleted, private and untitled videos from its results.

## Requirements

### Requirement: Unavailable entries are removed from the catalog
The catalog handler SHALL NOT return an entry whose raw yt-dlp title, trimmed and compared case-insensitively, is `[Deleted video]` or `[Private video]`.

#### Scenario: Deleted video in a playlist
- **WHEN** a playlist contains entries titled `Video A`, `[Deleted video]` and `Video C`
- **THEN** the catalog response contains `Video A` and `Video C` and no entry for the deleted video

#### Scenario: Private video in a playlist
- **WHEN** a playlist contains an entry titled `[Private video]`
- **THEN** the catalog response does not contain it

### Requirement: Untitled entries are removed
The catalog handler SHALL NOT return an entry whose raw yt-dlp title is missing or empty, and the `Unknown Title` fallback SHALL NOT be used to keep such an entry in the catalog.

#### Scenario: Entry without a title
- **WHEN** yt-dlp returns an entry with no `title` field
- **THEN** the catalog response does not contain an `Unknown Title` item for it

### Requirement: Available entries are unchanged
Entries that are not removed SHALL be returned with the same fields as before and in the same relative order.

#### Scenario: Order preserved
- **WHEN** a playlist contains `Video A`, `[Private video]`, `Video B` in that order
- **THEN** the catalog response lists `Video A` before `Video B` with unchanged metas

#### Scenario: Single video or channel
- **WHEN** the catalog is a single video or a channel page wrapped as one item and its title is a normal title
- **THEN** that one item is returned

### Requirement: Empty catalog when nothing is available
When every entry in a page is removed, the catalog handler SHALL return `{ metas: [] }`.

#### Scenario: Whole page unavailable
- **WHEN** every entry in a page is titled `[Deleted video]` or `[Private video]`
- **THEN** the response is `{ metas: [] }`
