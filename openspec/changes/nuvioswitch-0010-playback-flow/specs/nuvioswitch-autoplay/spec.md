## ADDED Requirements

### Requirement: Player settings defaults
The app SHALL keep per-profile player settings with these defaults:

| Setting | Default |
|---|---|
| Autoplay mode | Manual |
| Regex | empty |
| Selected add-ons | none (meaning all) |
| Autoplay timeout | 10 s |
| Reuse last link | on, kept for 1 hour |
| Reuse binge group | off |
| Prefer binge group for next episode | on |
| Autoplay next episode | off |
| Fall back to normal selection for the next episode | on |
| Next-episode threshold | percentage, 99% (or 2 minutes before the end when set to minutes) |
| "Still watching?" | off, after 3 episodes |
| Skip intro button | on |
| Auto-skip segment types | none |

#### Scenario: Fresh install
- **WHEN** the app starts with no `player_settings.json`
- **THEN** autoplay is Manual, the next-episode card appears at 99%, and the skip-intro button is enabled

### Requirement: Autoplay effective rule
Autoplay SHALL count as enabled when any of these is true:
- reuse last link is on;
- reuse binge group and prefer binge group for the next episode are both on;
- the mode is First stream;
- the mode is Regex match and the regex is valid and contains at least one letter or digit.

#### Scenario: Invalid regex
- **WHEN** the mode is Regex match, the pattern is `(((`, and reuse last link and binge-group reuse are off
- **THEN** autoplay is not enabled and the stream picker behaves as in Manual mode

### Requirement: Playable stream rule
For autoplay, a stream SHALL be playable only when all of these hold:
- it is not an `externalUrl`-only stream;
- it is not shown as unsupported by the stream picker (for example a torrent with no debrid);
- it has a playable HTTP(S) URL.

#### Scenario: Skip an external link
- **WHEN** the first stream in add-on order only has an `externalUrl`
- **THEN** First stream mode picks the next playable stream instead

### Requirement: Stream selection
Given the streams ordered as the stream picker orders them, autoplay SHALL select as follows:
1. Keep only streams from the selected add-ons, or all streams when none are selected.
2. If a preferred binge group is set and binge-group preference applies, return the first playable stream with that `bingeGroup`. When only binge-group matching is allowed, return nothing if there is no match.
3. In Manual mode, return nothing.
4. In First stream mode, return the first playable stream.
5. In Regex match mode, compile the pattern ignoring case. Build exclusion words from every negative lookahead group `(?!...(a|b))`.
6. Return the first playable stream whose text contains a regex match and no whole-word exclusion match. That text is add-on name, name, title, description and URL, joined by spaces.

#### Scenario: Regex with exclusion
- **WHEN** the regex is `^(?!.*(CAM|TS)).*1080p` and the streams are "Movie 1080p CAM" then "Movie 1080p WEB-DL"
- **THEN** "Movie 1080p WEB-DL" is selected

#### Scenario: Binge group wins in Manual mode
- **WHEN** the mode is Manual, binge-group preference applies for the next episode, and a stream has the same `bingeGroup` as the episode just played
- **THEN** that stream starts without the picker being shown

### Requirement: Autoplay in the stream picker
When autoplay is enabled and the stream picker opens, the picker SHALL:
- show streams as they arrive and run selection each time an add-on responds;
- start playback as soon as a stream is selected;
- when the timeout passes with nothing selected, stop autoplaying and leave the picker open for manual choice.

The user SHALL be able to cancel autoplay with B, which keeps the picker open.

#### Scenario: Timeout with no match
- **WHEN** the mode is Regex match and no stream matches within 10 seconds
- **THEN** the picker stays open with all streams listed and nothing starts

### Requirement: Reuse last link
After a stream has played its first frame, the app SHALL remember it per profile for the content key: the episode's `videoId`, or the movie's `contentId`. It stores URL, stream name, request headers, filename, video hash and size, and binge group. When reuse last link is on and a remembered link is younger than the cache window (default 1 hour), opening that item SHALL play the remembered link straight away, without querying add-ons.

#### Scenario: Reopen within an hour
- **WHEN** the user stops a film and plays it again 20 minutes later
- **THEN** the same stream starts immediately, at the resume position

#### Scenario: Expired link
- **WHEN** the remembered link is 2 hours old
- **THEN** the stream picker opens and queries add-ons as usual

### Requirement: Next-episode resolution
The next episode SHALL be the episode after the current one in `Meta.videos`, ordered by season then episode, ignoring videos without both numbers. For absolute-numbered content with no season, videos SHALL be ordered by episode number alone.

#### Scenario: Last episode of a season
- **WHEN** the current episode is S1E10 and S2E1 exists
- **THEN** the next episode is S2E1

### Requirement: Next-episode prompt
During playback of an episode that has a next episode, the player SHALL show a next-episode card when the next-episode threshold is reached:
- **Percentage mode:** the percentage is clamped to 97–100. **Minutes mode:** the minutes are clamped to 0–3.5.
- **Known outro segment:**
  - if more than the threshold remains after the outro ends, the card appears at the threshold;
  - otherwise it appears at the start of the outro;
  - it never appears over a post-credits scene: an explicit IntroDB post-credits segment, or more than 5 s of video after the outro.

The card SHALL offer "Play next" and dismiss with B. When "Autoplay next episode" is on, a 10-second countdown on the card SHALL start the next episode. At the natural end of an episode, the next episode SHALL start when autoplay-next is on; otherwise the card stays shown.

#### Scenario: Card at 99 percent
- **WHEN** a 45-minute episode with no skip data reaches 99%
- **THEN** the next-episode card appears

#### Scenario: Outro near the end
- **WHEN** the outro runs from 43:00 to 44:58 of a 45:00 episode in percentage mode at 99%
- **THEN** the card appears at 43:00

### Requirement: Playing the next episode
"Play next", or the countdown finishing, SHALL resolve a stream for the next episode:
- **Manual mode, with "Autoplay next episode" on or binge-group preference on:** auto-select; restricted to binge-group matches only when fallback is off, or when autoplay-next is off.
- **Manual mode, with neither on:** open the next episode's stream picker.
- **Other modes:** use the normal selection with the current stream's binge group preferred.

When no stream is selected within the timeout, the stream picker SHALL open for the next episode. The current episode SHALL be marked watched before switching.

#### Scenario: Same source for next episode
- **WHEN** the user presses "Play next" in Manual mode with binge-group preference on and the same add-on returns a stream with the same `bingeGroup`
- **THEN** that stream starts and no picker is shown

### Requirement: Still watching prompt
When "still watching?" is on, autoplay-next is on, the next episode has aired, and the number of episodes started automatically in a row has reached the threshold (default 3), the player SHALL pause before starting the next episode. It SHALL show "Are you still watching?" with a 60-second countdown. "Continue" resets the count and plays on; when the countdown ends, the player exits. Any user-initiated playback SHALL reset the count.

#### Scenario: Third automatic episode
- **WHEN** two episodes have auto-played in a row and a third is about to start
- **THEN** the prompt appears and the third episode only starts if the user presses "Continue"
