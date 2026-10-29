## ADDED Requirements

### Requirement: Preferred audio and subtitle languages
On each load, the player SHALL set mpv's `alang` and `slang` from the profile's preferred and secondary languages, preferred first. The default audio preference "Device" SHALL resolve to the console's system language (on desktop, the OS locale). The default subtitle preference SHALL be English when the system language is not set. The player SHALL also:
- prefer forced subtitle tracks when "forced subtitles" is on (default on);
- set `sub-filter-sdh=yes` when "strip SDH" is on (default on).

A track the user picks by hand SHALL win for the rest of that playback.

#### Scenario: Japanese audio, English subtitles
- **WHEN** preferred audio is Japanese, preferred subtitles are English, and the file has jpn and eng audio and eng subtitles
- **THEN** playback starts with the Japanese audio and English subtitles selected

### Requirement: Subtitle styling
The player SHALL apply the profile's subtitle style through mpv properties on load and whenever the style changes:

| Setting | Default | Range |
|---|---|---|
| Size (`sub-scale`) | 120% | 50–200% |
| Vertical offset (`sub-pos` / `sub-margin-y`) | 5% | −20–50% |
| Text colour (`sub-color`) | white | |
| Background colour (`sub-back-color`) | transparent | |
| Outline (`sub-outline-color`, `sub-outline-size`) | on, black, width 2 | width 1–5 |
| Bold (`sub-bold`) | off | |

ASS subtitles SHALL keep their own styling (`sub-ass-override=no`).

#### Scenario: Larger subtitles
- **WHEN** the subtitle size is set to 150%
- **THEN** the player sets `sub-scale` so that text subtitles render 1.5× the base size, and ASS subtitles are unchanged
