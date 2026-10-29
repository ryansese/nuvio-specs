## MODIFIED Requirements

### Requirement: Playability
A stream SHALL be playable when it has an HTTP or HTTPS URL (`url`, else `externalUrl`) that is not a magnet link or `.torrent` URL. Torrent-only streams (an info hash, magnet link or torrent URL and no HTTP URL) SHALL be playable through a connected debrid service as described in "Debrid-backed torrent playback"; when no debrid service is connected they SHALL be shown dimmed with a "Needs debrid" label. Streams with only an `externalUrl` web link, or only a `ytId`, SHALL be shown dimmed with an "Unsupported" label. Selecting a non-playable stream SHALL show a short explanation and SHALL NOT open the player.

#### Scenario: HTTP stream
- **WHEN** a stream has `url` `https://cdn.example/movie.mp4`
- **THEN** it is playable

#### Scenario: Torrent only
- **WHEN** a stream has `infoHash` and `fileIdx` but no `url`, and no debrid service is connected
- **THEN** it is shown with "Needs debrid" and selecting it shows a message instead of playing

## ADDED Requirements

### Requirement: Debrid-backed torrent playback
When at least one debrid service is connected for the active profile, torrent-only streams (an info hash, magnet link or torrent URL, and no HTTP URL) SHALL be shown as selectable instead of with "Needs debrid". They SHALL carry a badge with the service's short name (TB, PM, RD) and the cache state. Selecting one SHALL show a resolving indicator, resolve it through nuvioswitch-debrid, and open the player with the resulting URL and the stream's other hand-off data. Resolution failures SHALL show a short message (Not cached, Link expired, Service error) and leave the stream list open. With no debrid service connected, the 0007 "Needs debrid" behaviour SHALL apply unchanged.

Streams from add-ons that use `clientResolve` SHALL be resolved through the service named in `clientResolve.service` when the user has a key for it. Otherwise they SHALL be treated as torrent-only.

#### Scenario: Torrent playable with Torbox
- **WHEN** Torbox is connected and the user selects a torrent stream marked Cached
- **THEN** a resolving indicator shows, then the player opens with the Torbox URL

#### Scenario: No debrid
- **WHEN** no debrid service is connected
- **THEN** torrent-only streams are still dimmed with "Needs debrid"

#### Scenario: Resolve fails
- **WHEN** resolution returns Not cached
- **THEN** the stream list stays open with "Not cached on Torbox" shown, and the stream's badge updates to Not cached
