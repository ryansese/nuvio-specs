## ADDED Requirements

### Requirement: Main window and tab navigation
On launch the app SHALL open a single window titled "NuvioSwitch" containing a sidebar with the tabs Home, Search, Library, Add-ons and Settings, in that order, with Home selected. The order follows NuvioTV's sidebar (`ui/navigation`, `MainActivity.kt`), with Add-ons promoted from a Settings sub-screen to a tab.

#### Scenario: App launches to Home
- **WHEN** the user starts NuvioSwitch on Switch or desktop
- **THEN** the window shows the sidebar with the five tabs and the Home tab's content

#### Scenario: Switching tabs with the gamepad
- **WHEN** the user moves focus to the sidebar with the D-pad or left stick and selects another tab with A
- **THEN** that tab's content is shown and focus can move right into it

### Requirement: Placeholder content per tab
Until its feature change lands, each tab SHALL show a centred placeholder with the tab's name and a one-line note that the feature is not available yet. The Settings tab SHALL also show the app name and version.

#### Scenario: Placeholder shown
- **WHEN** the user opens the Library tab in this version
- **THEN** a placeholder reading "Library" with a "coming soon" note is shown, and nothing crashes

#### Scenario: Version visible
- **WHEN** the user opens the Settings tab
- **THEN** it shows "NuvioSwitch <version>" matching the build's version

### Requirement: Exit behaviour
The app SHALL exit when the user presses + (Plus) on Switch, or closes the window or presses Escape on desktop. B on a root tab SHALL NOT exit the app.

#### Scenario: Exit on Switch
- **WHEN** the user presses + anywhere in the shell
- **THEN** the app closes and returns to the homebrew menu

#### Scenario: B does not exit
- **WHEN** the user presses B with focus on a tab's content
- **THEN** focus moves back to the sidebar and the app keeps running

### Requirement: Nuvio dark theme
The app SHALL use borealis's dark theme variant, with colours overridden to match NuvioTV's default dark theme (`ui/theme/PrimitiveTokens.kt`): background `#0D0D0D`, sidebar `#1A1A1A`, primary text white, secondary text `#B3B3B3`, and a white focus highlight.

#### Scenario: Colours applied
- **WHEN** the app is running
- **THEN** the window background is `#0D0D0D`, the sidebar `#1A1A1A`, and the focused item has a white highlight border

### Requirement: Localised strings
All user-visible strings in the shell SHALL come from `resources/i18n/en-US/` and not be hard-coded, so later translations only add JSON files.

#### Scenario: Tab labels from i18n
- **WHEN** a tab label is changed in `resources/i18n/en-US/app.json` and the app is rebuilt
- **THEN** the sidebar shows the new label without code changes
