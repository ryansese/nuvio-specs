## ADDED Requirements

### Requirement: Account section
When account features are available, Settings SHALL have an Account section, shown only on the primary profile. Signed out, it shows a Sign in action that opens the QR sign-in screen. Signed in, it shows the account email (when the session has one) and Sign out.

#### Scenario: Sign in from Settings
- **WHEN** the user skipped the account gate earlier and opens Settings → Account → Sign in
- **THEN** the QR sign-in screen opens, and after approval the section shows the signed-in state

#### Scenario: Hidden on secondary profiles
- **WHEN** profile 2 is active
- **THEN** Settings doesn't show the Account section

### Requirement: Profiles section
Settings SHALL have a Profiles section on the primary profile for creating, renaming, recolouring and deleting profiles. When signed in, it also offers setting, changing or removing a profile's PIN.

#### Scenario: Set a PIN
- **WHEN** the signed-in user picks profile 2 → Set PIN and enters 1234 twice
- **THEN** `set_profile_pin` is called for profile 2, and the selection screen shows it as locked
