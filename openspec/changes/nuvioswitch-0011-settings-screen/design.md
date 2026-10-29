## Context

After 0010, settings live in two places:
- `player_settings.json` (0010), using NuvioTV's `PlayerSettings` names;
- a few layout flags read with defaults by 0004 and 0009.

NuvioTV's settings UI has three styles (Classic, Zen, Horizon), a rail grouped YOU / LOOK / WATCH / SERVICES / SYSTEM, and dozens of screens. All paths below are under `app/src/main/java/com/nuvio/tv/`: `ui/screens/settings/SettingsScreen.kt`, `SettingsCatalog.kt`, `PlaybackSettingsModel.kt` and `ThemeSettingsScreen.kt`.

Borealis provides the setting cells we need. All are under `library/include/borealis/views/cells/`:
- `BooleanCell` (`cell_bool.hpp`), with `init(title, isOn, callback)`;
- `SelectorCell` (`cell_selector.hpp`), with `init(title, data, selected, callback)`, which opens a dropdown;
- `InputCell` (`cell_input.hpp`), which uses swkbd on Switch;
- `SliderCell` (`cell_slider.hpp`), `RadioCell` (`cell_radio.hpp`) and `DetailCell` (`cell_detail.hpp`);
- `Dialog` (`views/dialog.hpp`) for confirmations.

## Goals / Non-Goals

**Goals:**
- One settings screen covering every setting that exists by 0010, built from stock borealis cells.
- Live apply where cheap: theme, layout toggles and the subtitle preview.

**Non-Goals:**
- NuvioTV's three settings styles, the Essential/Advanced experience mode, custom gradient themes, supporter themes (Gold, Jade, Rose Gold, Arctic Blue, Graphite), font choice, app language selection and the startup splash.
- Account, profiles, integrations and debrid sections; each comes with its own change (0013–0018). The section list is a data table so they can add entries.
- Buffer and network tuning beyond the mpv cache size.

## Decisions

**1. Layout: `Box` row with a section list and a `ScrollingFrame` panel, rather than borealis's `TabFrame` again.**
- A nested `TabFrame` inside the main `TabFrame` gives a second sidebar that looks like the main one and confuses focus.
- The section list is a vertical list of `RadioCell`-style items. Each section is a class that builds its cells on focus (`SettingsSection::build(Box*)`).
- Alternative: a nested `TabFrame`. Rejected for the focus and visual reasons above.

**2. Section registry.**
- `src/settings/section_registry.cpp` lists sections in order as `{id, i18nKey, factory}`.
- Later changes (account, Trakt, debrid) add a line rather than editing the screen.

**3. Settings model and change notification.**
- `AppSettings` (new) and `PlayerSettings` (0010) each have `load()`, `save()` and a borealis `Event<>` fired on change.
- Theme, Home and the player subscribe; for example, Home re-renders when Continue Watching or poster labels change.
- Saves use 0002's atomic write on the worker thread.

**4. Theme palettes.**
- Free palettes are copied from NuvioTV's `ThemeColors.kt` and `PrimitiveTokens.kt`: secondary, focus ring and focus background for White, Crimson, Ocean, Violet, Emerald and Amber, Rose.
- They're stored as a table in `src/core/settings/theme_palettes.cpp`.
- Applying a palette calls `Theme::getDarkTheme().addColor()` for `brls/highlight/color1`, `color2`, `brls/accent` and `brls/sidebar/active_item`, then invalidates the views.

**5. Regex entry and presets.**
- Presets are listed in a `SelectorCell`, with a "Custom…" entry opening an `InputCell`/swkbd. The keyboard's maximum length is raised to 256.
- Validation reuses 0010's `isRegexSelectionConfigured`.

**6. Subtitle preview.**
- A `Label` styled with the chosen text colour, background, bold and an approximate size. NanoVG can't draw mpv's outline exactly, so the outline is approximated with a text shadow.
- It's a preview of colours and size, not pixel-exact.

**7. Video mapping.**
- Hardware decoding: Auto-safe → `hwdec=auto-safe`; Hardware → the hardware decoder 0008 configured for the platform (on Switch, the decoder its mpv build supports); Off → `hwdec=no`.
- Cache size → `demuxer-max-bytes` (and the same value for `demuxer-max-back-bytes`, as NuvioTV does).

**8. About QR code.**
- A small single-file QR encoder (Nayuki's `qrcodegen`, MIT) is vendored in `src/third_party/` and drawn with NanoVG rectangles.
- 0013 reuses it for the sign-in QR.

## Risks / Trade-offs

- [Too many cells in one scrolling panel are slow to build on Switch] → Sections are built lazily when selected and destroyed when left.
- [swkbd isn't available in applet mode (album launch)] → `InputCell` falls back to borealis's on-screen keyboard. Users are told to launch with title takeover (documented in 0012's README).
- [A 256 MiB cache on Switch with large files can exhaust memory] → Show a note under the option. Keep 64 MiB as the default, as NuvioTV does.
- [Changing the theme while deep in a stack leaves stale colours on some views] → Invalidate the whole view tree, and confirm in the desktop and hardware checks.

## Open Questions

- Should the supporter themes (Gold, Jade, Rose Gold, Arctic Blue, Graphite) be offered too? NuvioTV restricts them to supporters, so this change leaves them out.
