## ADDED Requirements

### Requirement: Debrid section
Settings SHALL have a per-profile Debrid section listing Torbox, Premiumize and Real-Debrid:
- each shows Connected or Not connected;
- Connect offers device login where available, or Enter API key;
- Disconnect clears the stored key;
- a Preferred resolver choice is shown when more than one service that can resolve torrents is connected.

Keys SHALL be masked when shown.

#### Scenario: Connect by API key
- **WHEN** the user picks Real-Debrid → Enter API key and types a valid key
- **THEN** the key is validated, saved and shown masked, and Real-Debrid shows as connected
