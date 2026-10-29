## ADDED Requirements

### Requirement: Simkl and MDBList sections
Settings SHALL have a Simkl section when Simkl is configured, and an MDBList section. Both work per profile.
- Simkl: Connect, Connected, Disconnect.
- MDBList: Connect, Connected, Disconnect (when `MDBLIST_CLIENT_ID` is set), an API key field entered with the on-screen keyboard and checked against `GET /user?apikey=…`, and a "Show MDBList ratings" toggle that is off by default, matching NuvioTV's `mdblist_enabled` default.

#### Scenario: Enter API key
- **WHEN** the user enters a valid MDBList API key and turns on ratings
- **THEN** the key is saved for the profile and details screens show MDBList ratings
