## ADDED Requirements

### Requirement: Settings structure
The Settings tab SHALL show a list of sections on the left: Appearance, Layout, Playback, Skip segments, Audio & subtitles, Video, Storage, About. The selected section's settings SHALL show on the right. Focus SHALL move between the list and the panel with the D-pad, and B in the panel SHALL return focus to the section list.

#### Scenario: Navigate to a section
- **WHEN** the user opens Settings and selects "Playback"
- **THEN** the right panel lists the playback settings and focus can move into it

### Requirement: Settings are saved immediately per profile
Each change SHALL be written to the active profile's settings file as soon as it is made, and SHALL be applied without restarting the app:
- appearance and layout go to `app_settings.json`;
- playback, skip, audio, subtitle and video settings go to `player_settings.json`.

Video settings SHALL apply from the next playback.

#### Scenario: Persist across restart
- **WHEN** the user turns poster labels off and restarts the app
- **THEN** poster labels are still off

### Requirement: Appearance settings
The Appearance section SHALL offer an accent theme selector with White (default), Crimson, Ocean, Violet, Emerald, Amber and Rose. Choosing a theme SHALL update the focus highlight, sidebar active item and accent colour immediately.

#### Scenario: Change accent
- **WHEN** the user selects "Ocean"
- **THEN** the focus highlight changes to the Ocean focus-ring colour at once, with no restart

### Requirement: Layout settings
The Layout section SHALL offer three toggles:
- "Continue Watching" (default on); when off, the Home row is hidden;
- "Poster labels" (default on); when off, poster titles are hidden in rows and grids;
- "Hide unreleased" (default off); when on, catalog items whose release info is in the future are hidden.

#### Scenario: Hide Continue Watching
- **WHEN** the user turns "Continue Watching" off
- **THEN** Home no longer shows the Continue Watching row, and progress is still saved

### Requirement: Playback settings
The Playback section SHALL offer:
- **Autoplay:**
  - autoplay mode (Manual, First stream, Regex match);
  - a regex field, shown only for Regex match, with presets such as "1080p, no CAM/TS" and "4K first" plus custom text entry through the software keyboard;
  - autoplay add-ons (multi-select from installed add-ons; none selected means all);
  - autoplay timeout (5, 10, 15, 30 s or unlimited);
  - reuse last link with its window (1, 6, 12 or 24 h);
  - reuse binge group, and prefer binge group for the next episode.
- **Next episode:**
  - autoplay next episode, and fall back to normal selection;
  - next-episode threshold mode (percentage or minutes before end) and value (97–100% or 0–3.5 min);
  - "Still watching?" and its episode count (2–6).

An invalid custom regex SHALL be refused, with an error message, and the previous value kept.

#### Scenario: Invalid regex
- **WHEN** the user enters `(((` as the autoplay regex
- **THEN** an "Invalid pattern" message is shown and the stored regex is unchanged

#### Scenario: Regex field visibility
- **WHEN** the autoplay mode is Manual
- **THEN** the regex field is not shown

### Requirement: Skip segment settings
The Skip segments section SHALL offer a "Skip button" toggle (default on), and an auto-skip multi-select for Intro, Recap, Outro and Movie credits (default none).

#### Scenario: Enable auto-skip intro
- **WHEN** the user selects Intro in auto-skip
- **THEN** the next playback skips intros automatically

### Requirement: Audio and subtitle settings
The Audio & subtitles section SHALL offer:
- **Audio language:** preferred (Device default, or a list of common languages) and secondary (None or a language).
- **Subtitle language:** preferred and secondary.
- **Toggles:** forced subtitles (default on) and strip SDH (default on).
- **Subtitle style:**
  - size 50–200% in steps of 10 (default 120);
  - vertical offset −20–50% (default 5);
  - text colour and background colour from a palette (defaults white and transparent);
  - outline on/off (default on), outline colour (default black) and outline width 1–5 (default 2);
  - bold (default off).
- A preview line rendered with the current text style.
- "Reset subtitle style".

#### Scenario: Preview updates
- **WHEN** the user changes the subtitle text colour to yellow
- **THEN** the preview line text turns yellow immediately

### Requirement: Video settings
The Video section SHALL offer hardware decoding (Auto-safe default, Hardware, Off) mapped to mpv `hwdec` values, and stream cache size (64 MiB default, 128 MiB, 256 MiB) mapped to `demuxer-max-bytes`.

#### Scenario: Turn off hardware decoding
- **WHEN** the user sets hardware decoding to Off and starts a stream
- **THEN** mpv is configured with `hwdec=no` for that playback

### Requirement: Storage actions
The Storage section SHALL show the current size of the image cache and HTTP cache, and offer:
- "Clear image cache" and "Clear HTTP cache";
- "Show hidden Continue Watching items again";
- "Reset all settings", which restores the defaults for the active profile's app and player settings. It SHALL NOT touch the library, progress or add-ons, and SHALL ask for confirmation first.

#### Scenario: Reset settings
- **WHEN** the user confirms "Reset all settings"
- **THEN** all settings return to their defaults, and the library, watch progress and installed add-ons are unchanged

### Requirement: About section
The About section SHALL show:
- "NuvioSwitch <version>";
- the project URL, both as text and as a QR code;
- a credit line naming NuvioTV and NuvioMedia as the original app;
- a scrollable list of third-party licences.

#### Scenario: Version shown
- **WHEN** the user opens About on build 0.1.0
- **THEN** it shows "NuvioSwitch 0.1.0"
