## ADDED Requirements

### Requirement: Saved accent theme applied at launch
On launch the app SHALL read the active profile's accent theme from `app_settings.json` and apply its colours (focus highlight, accent, sidebar active item) before the first frame is drawn. It SHALL fall back to White when the setting is missing or unknown.

#### Scenario: Theme survives restart
- **WHEN** the user picked "Emerald" and restarts the app
- **THEN** the first screen already uses the Emerald focus colour

#### Scenario: Unknown value
- **WHEN** `app_settings.json` contains an unknown theme name
- **THEN** the app starts with the White theme and does not crash
