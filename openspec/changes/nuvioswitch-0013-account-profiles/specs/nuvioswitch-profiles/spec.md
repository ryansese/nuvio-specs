## ADDED Requirements

### Requirement: Local profiles
The app SHALL support up to 6 profiles, as `core/profile/ProfileManager.kt` does. Each profile has an integer id, a name, an avatar colour (hex) and a "use primary add-ons" flag. Profile 1 always exists, is the primary profile, and can't be deleted. A new profile takes the lowest free id from 2 to 6. Profiles SHALL work without an account. Each profile's data lives in its own `profiles/<id>/` stores (nuvioswitch-storage).

#### Scenario: Create a profile
- **WHEN** profiles 1 and 2 exist and the user creates a profile named "Kids"
- **THEN** it gets id 3, has its own empty library and progress, and appears on the profile screen

#### Scenario: Limit reached
- **WHEN** 6 profiles exist
- **THEN** the Add profile action is hidden

#### Scenario: Delete a profile
- **WHEN** the user deletes profile 3
- **THEN** its local stores are removed, and when signed in `rpc/sync_delete_profile_data` is called with `{p_profile_id: 3}`

#### Scenario: Shared add-ons
- **WHEN** profile 2 has "use primary add-ons" turned on
- **THEN** it reads and writes profile 1's add-on list

### Requirement: Profile selection at launch
The app SHALL show a "Who's watching?" screen after the account gate when there is more than one profile or the active profile has a PIN. Otherwise it SHALL open profile 1 directly. Switching profiles SHALL reload the per-profile data and return to Home.

#### Scenario: Single profile without PIN
- **WHEN** only profile 1 exists and it has no PIN
- **THEN** the app opens Home without the selection screen

#### Scenario: Switch profile from the sidebar
- **WHEN** more than one profile exists and the user picks the profile item in the sidebar and selects another profile
- **THEN** Home reloads with that profile's Continue Watching and add-ons

### Requirement: Profile PIN lock
When signed in, the app SHALL let each profile have an optional 4-digit PIN, managed through the server as in `core/sync/ProfileSyncService.kt`:
- set: `rpc/set_profile_pin {p_profile_id, p_pin, p_current_pin?}`
- clear: `rpc/clear_profile_pin {p_profile_id, p_current_pin?}`
- verify: `rpc/verify_profile_pin {p_profile_id, p_pin}`, returning `unlocked` and `retry_after_seconds`
- lock states: `rpc/sync_pull_profile_locks`, cached locally so the selection screen knows which profiles are locked

PINs are entered on a gamepad-friendly 4-digit pad.

#### Scenario: Unlock with the right PIN
- **WHEN** the user picks a locked profile and enters the correct PIN
- **THEN** `verify_profile_pin` returns `unlocked: true` and the profile opens

#### Scenario: Wrong PIN rate-limited
- **WHEN** `verify_profile_pin` returns `unlocked: false` with `retry_after_seconds: 30`
- **THEN** the pad shows an error and stays disabled for 30 seconds

#### Scenario: Change a PIN that requires the current one
- **WHEN** `set_profile_pin` fails with "Current PIN is required"
- **THEN** the app asks for the current PIN and retries with `p_current_pin`

#### Scenario: Locked profile offline
- **WHEN** a profile is locked and the server can't be reached
- **THEN** the profile stays locked and the screen says a connection is needed to unlock it

### Requirement: Profile list sync
When signed in, the app SHALL push the profile list with `rpc/sync_push_profiles {p_client_max_profiles: 6, p_profiles: [{profile_index, name, avatar_color_hex, uses_primary_addons, uses_primary_plugins, avatar_id, avatar_url, profile_background_id, profile_background_url}]}` after every change. After sign-in and at launch it SHALL pull with `rpc/sync_pull_profiles`, retrying up to 3 times right after sign-in as NuvioTV does. Remote profiles replace local ones by `profile_index`. Fields the Switch doesn't use (avatar images, backgrounds, plugins) SHALL be preserved, not cleared.

#### Scenario: Profiles from another device
- **WHEN** the user signs in on the Switch with an account that has three profiles created on NuvioTV
- **THEN** the selection screen shows those three profiles with their names and colours

#### Scenario: Preserving unknown fields
- **WHEN** a remote profile has an `avatar_url` and the user renames it on the Switch
- **THEN** the push keeps the same `avatar_url`
