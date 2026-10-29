## ADDED Requirements

### Requirement: Trakt section
When Trakt is configured in the build, Settings SHALL have a Trakt section for the active profile. Not connected, it shows Connect, which opens the device-code screen. Connected, it shows "Connected as <username>" and Disconnect.

#### Scenario: Disconnect
- **WHEN** the user picks Disconnect in the Trakt section
- **THEN** the token is revoked, the section shows Connect, and the Library's Trakt tab is hidden
