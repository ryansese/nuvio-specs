## ADDED Requirements

### Requirement: Account features depend on service config
Account features SHALL be available only when the build has a non-empty `NUVIO_SUPABASE_URL` and `NUVIO_SUPABASE_ANON_KEY`. Without them, the app SHALL skip the sign-in gate, hide account entry points and run local-only.

#### Scenario: Build without config
- **WHEN** the app is built without `config.local.json` and started
- **THEN** no sign-in screen appears, Settings has no Account section, and browsing and playback work as in v0.1

### Requirement: QR device sign-in
The app SHALL sign in with NuvioTV's device-login flow (`core/auth/AuthManager.kt`):
1. Generate a nonce of 24 random bytes, base64url-encoded without padding.
2. `POST {NUVIO_SUPABASE_URL}/rest/v1/rpc/start_device_login_session` with `{p_device_nonce, p_redirect_base_url: DEVICE_LOGIN_WEB_BASE_URL, p_device_type, p_device_name}`.
3. Show a QR code for `verification_uri_complete`, plus `user_code` and `verification_uri` as text, with a countdown to `expires_at`.
4. Poll `rpc/poll_tv_login_session` with `{p_code: device_code, p_device_nonce}` every `poll_interval_seconds` (at least 2 s).
5. When the status is `approved`, `POST {NUVIO_SUPABASE_URL}/functions/v1/tv-logins-exchange` with `{code, device_nonce}` and store the returned `access_token` and `refresh_token`.

All Supabase requests SHALL send `apikey: <anon key>`, and authenticated ones also `Authorization: Bearer <access token>`.

#### Scenario: Successful sign-in
- **WHEN** the user scans the QR code, approves on the phone, and the next poll returns `approved`
- **THEN** the app exchanges the code, stores the tokens, shows the account as signed in, and continues to profile selection

#### Scenario: Legacy backend
- **WHEN** `start_device_login_session` fails because the function doesn't exist
- **THEN** the app calls `rpc/start_tv_login_session` with `{p_device_nonce, p_redirect_base_url: TV_LOGIN_WEB_BASE_URL, p_device_name}` and uses its `code` and `web_url` for polling and the QR code

#### Scenario: Code expires
- **WHEN** the poll returns `expired`, or the countdown reaches zero, before approval
- **THEN** polling stops and the screen shows that the code expired, with a Refresh action that starts a new session

#### Scenario: Cancelled or already used
- **WHEN** the poll returns `cancelled` or `used`
- **THEN** polling stops and the user is offered a new code

#### Scenario: Leaving the screen
- **WHEN** the user backs out of the sign-in screen while polling
- **THEN** polling stops and no exchange is made

### Requirement: Session persistence and refresh
The session SHALL be stored in a global store and restored on launch. When the access token is near expiry, or a request fails with an expired-JWT error, the app SHALL refresh once with `POST /auth/v1/token?grant_type=refresh_token`, serialising concurrent refreshes, and retry the request. If the refresh token is rejected as invalid, the app SHALL sign out locally. A network failure during refresh SHALL keep the session.

#### Scenario: Restart while signed in
- **WHEN** the user restarts the app after signing in
- **THEN** the app is still signed in without showing the sign-in screen

#### Scenario: Invalid refresh token
- **WHEN** the refresh endpoint responds with an invalid or not-found refresh token
- **THEN** the session is cleared and Settings shows the account as signed out

#### Scenario: Offline at launch
- **WHEN** the app starts signed in without network access
- **THEN** the session is kept and the app works with local data

### Requirement: Effective user and device registration
After sign-in and on each launch while signed in, the app SHALL resolve the effective user ID with `rpc/get_sync_owner`, falling back to its own user ID if that fails, and register the device with `rpc/register_current_device`, as `core/auth/DeviceSessionRegistration.kt` does.

#### Scenario: Linked device
- **WHEN** `get_sync_owner` returns a different user ID from the signed-in one
- **THEN** that ID is used as the owner for direct table reads in sync

### Requirement: First-launch account gate
On the first launch with service config present, the app SHALL show the QR sign-in screen with a **Continue without account** action. Choosing it SHALL be remembered, and the gate SHALL NOT show again. Sign-in stays available from Settings.

#### Scenario: Skip account
- **WHEN** the user picks Continue without account
- **THEN** the app opens the main shell, and on the next launch goes straight there

### Requirement: Sign out
Settings SHALL offer Sign out. Signing out SHALL clear the stored session and the sync state, and SHALL keep the local library, progress and add-ons.

#### Scenario: Sign out keeps local data
- **WHEN** the user signs out
- **THEN** Settings shows the account as signed out, and the library and Continue Watching still show local items
