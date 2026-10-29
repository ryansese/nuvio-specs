## 1. Settings model

- [ ] 1.1 Add `src/core/settings/app_settings.{hpp,cpp}` (accent theme, Continue Watching, poster labels, hide unreleased), per-profile `app_settings.json`, defaults, change event
- [ ] 1.2 Add a change event to 0010's `PlayerSettings` and a "reset to defaults" for both models
- [ ] 1.3 Add `theme_palettes.cpp` with the seven free NuvioTV palettes and `applyTheme()`; apply the saved theme at launch before the first frame
- [ ] 1.4 Unit tests: defaults, round-trip, unknown theme falls back to White, reset leaves other stores untouched

## 2. Screen frame

- [ ] 2.1 Replace the Settings placeholder with `settings_tab.{hpp,cpp}` + XML: a section list on the left, a scrolling panel on the right, B returns to the list
- [ ] 2.2 Add `section_registry` and the `SettingsSection` interface with lazy build and teardown

## 3. Sections

- [ ] 3.1 Appearance: theme selector with live apply
- [ ] 3.2 Layout: Continue Watching, poster labels and hide unreleased toggles; Home and 0004 rows react to changes
- [ ] 3.3 Playback: autoplay mode, conditional regex (presets + custom swkbd input with validation), add-on multi-select, timeout, reuse last link + window, binge options, next-episode autoplay/fallback/threshold mode/value, still watching + count
- [ ] 3.4 Skip segments: skip button toggle and auto-skip multi-select
- [ ] 3.5 Audio & subtitles: languages, forced, strip SDH, style controls, preview line, reset style
- [ ] 3.6 Video: hardware decoding and cache size, mapped to the 0008 mpv options on next playback
- [ ] 3.7 Storage: cache sizes, clear image and HTTP cache, unhide Continue Watching items, reset all settings with a confirmation dialog
- [ ] 3.8 About: version, URL text and QR code (vendor `qrcodegen`), NuvioTV/NuvioMedia credit, licence list in `resources/licenses/`
- [ ] 3.9 Add all i18n strings to `resources/i18n/en-US/app.json`

## 4. Verification

- [ ] 4.1 `ctest --test-dir build_desktop` passes
- [ ] 4.2 Desktop: change every setting, restart and check it persisted; check the theme applies live and the subtitle preview updates
- [ ] 4.3 Desktop: an invalid regex is refused; a Manual-mode regex field is hidden; reset leaves library, progress and add-ons intact
- [ ] 4.4 Switch: build the `.nro`; on hardware check swkbd regex entry, focus movement between list and panel with the gamepad, and that the hwdec and cache settings take effect on the next playback
