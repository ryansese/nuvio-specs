# NuvioSwitch-0011 Settings screen

## Why

By the end of 0010 the app has many behaviours driven by settings: autoplay mode, next-episode threshold, skip segments, audio and subtitle languages, subtitle style, Continue Watching and poster labels. None of them can be changed from the app; they only have defaults in JSON files. The Settings tab is still the 0001 placeholder. Users need one place to change these before the first release (0012).

## What Changes

The Settings placeholder tab becomes a real settings screen. It has a left list of sections and a right panel of borealis setting cells, grouped like NuvioTV's settings rail (`ui/screens/settings/SettingsScreen.kt`, `SettingsCatalog.kt`) and trimmed to what NuvioSwitch supports:
- **Appearance:** accent theme (White, Crimson, Ocean, Violet, Emerald, Amber, Rose, from NuvioTV's free `ThemeColors.kt` palettes), applied immediately to the focus highlight and accent.
- **Layout:**
  - Continue Watching on or off (default on) and poster labels on or off (default on), as in NuvioTV's `LayoutPreferenceDataStore`;
  - "Hide unreleased" for catalog rows.
- **Playback:**
  - autoplay mode, autoplay regex (presets plus custom input), add-ons allowed for autoplay, autoplay timeout, reuse last link and its window, binge-group options;
  - autoplay next episode, the next-episode threshold (mode and value), still watching and its episode count.
- **Skip segments:** skip-intro button on or off, and auto-skip types (intro, recap, outro, movie credits).
- **Audio and subtitles:** preferred and secondary audio language, preferred and secondary subtitle language, forced subtitles, strip SDH, and subtitle style (size, vertical offset, text colour, background colour, outline on/off, outline colour and width, bold), with a live preview line.
- **Video:** mpv hardware decoding (Auto-safe, Hardware, Off) and stream cache size (64, 128 or 256 MiB, default 64), applied on the next playback.
- **Storage:** clear the image cache, clear the HTTP cache, clear Continue Watching hidden items, and reset all settings to defaults (with confirmation).
- **About:** app name and version, a link to the GitHub repository shown as text and a QR code, credit to NuvioTV and NuvioMedia, and open-source licences (borealis, libmpv, FFmpeg, libcurl, nlohmann/json, doctest, plus fonts and icons from NuvioTV).

Account and profile settings aren't part of this change; they come with 0013, and Trakt and others come with their own changes.

## Capabilities

### New Capabilities
- `nuvioswitch-settings`: the settings screen structure, every setting listed above with its options and default, how changes are stored and applied, the storage actions and the About section.

### Modified Capabilities
None. The extension below is an `ADDED` requirement on a capability introduced by an earlier, not-yet-archived change:
- `nuvioswitch-app-shell` (0001): adds the requirement that the chosen accent theme is applied on launch. The 0001 requirement that Settings shows the version is still met by the About section.

## Impact

- **Depends on:**
  - 0001 (shell, theme setup)
  - 0002 (storage, caches)
  - 0003 (installed add-ons list for autoplay scope)
  - 0004 (poster labels, hide unreleased)
  - 0008 (hwdec, cache options)
  - 0009 (Continue Watching toggle and hidden list)
  - 0010 (`player_settings.json` model)
- **Code (`repos/NuvioSwitch`):**
  - `src/tabs/settings_tab.*` and `src/settings/` (one view per section)
  - `src/core/settings/app_settings.*` (appearance and layout)
  - settings XML layouts and i18n strings
  - a small QR code generator for the About link (shared later with 0013)
- **Storage:**
  - new per-profile `app_settings.json` for appearance and layout;
  - player settings stay in 0010's `player_settings.json`.
- **Behaviour references (NuvioTV):**
  - `ui/screens/settings/SettingsScreen.kt`, `SettingsCatalog.kt`, `ThemeSettingsScreen.kt`, `PlaybackSettingsModel.kt`
  - `ui/theme/ThemeColors.kt`, `ui/theme/PrimitiveTokens.kt`
  - `data/local/LayoutPreferenceDataStore.kt`, `data/local/PlayerSettingsDataStore.kt`
