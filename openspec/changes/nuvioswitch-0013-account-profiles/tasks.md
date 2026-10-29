## 1. Service config

- [ ] 1.1 Add `config.example.json` with every key from the nuvioswitch-build spec, all empty, and keep `config.local.json` gitignored
- [ ] 1.2 Read `config.local.json` and `-D` overrides in CMake, generate `service_config.hpp`, and print disabled features
- [ ] 1.3 Add `ServiceConfig::accountEnabled()` and similar helpers used to hide features

## 2. Supabase client

- [ ] 2.1 Add `SupabaseClient` with `rpc`, `select`, `functionCall` and the auth token endpoints on top of the 0002 HTTP client
- [ ] 2.2 Add the session store (global `auth_session`) with load, save and clear
- [ ] 2.3 Add serialised token refresh, plus one refresh-and-retry when a JWT has expired, and classify invalid-token vs transient failures
- [ ] 2.4 Unit tests for refresh classification and the retry wrapper with a fake transport

## 3. QR sign-in

- [ ] 3.1 Vendor `qrcodegen` and add a `QrCodeView` drawn with nanovg
- [ ] 3.2 Implement the `QrLoginFlow` state machine (start with legacy fallback, poll, exchange, expire, cancel)
- [ ] 3.3 Unit tests for every poll status, the legacy fallback, interval clamping and expiry
- [ ] 3.4 Build the sign-in activity: QR code, user code, URL, countdown, Refresh, Continue without account
- [ ] 3.5 After sign-in: `get_sync_owner` (with fallback) and `register_current_device`

## 4. Profiles

- [ ] 4.1 Add `ProfileManager` (global `profiles` store, max 6, lowest free id, primary profile 1, observers on switch)
- [ ] 4.2 Point the per-profile stores from 0002 at the active profile, with "use primary add-ons" redirecting to profile 1
- [ ] 4.3 Add the "Who's watching?" activity and the sidebar profile item
- [ ] 4.4 Add the PIN pad and the set, clear and verify calls, with retry-after handling and the lock-state cache
- [ ] 4.5 Add profile sync: push on change, pull after sign-in (3 retries) and at launch, keep unknown fields, delete removed profiles' data
- [ ] 4.6 Unit tests for profile id allocation, the merge with unknown fields preserved, and the selection-screen rule

## 5. Launch flow and Settings

- [ ] 5.1 Add the launch sequence: account gate, then profile pull, then selection, then main shell
- [ ] 5.2 Add the Settings → Account section (Sign in, signed-in state, Sign out that keeps local data)
- [ ] 5.3 Add the Settings → Profiles section (create, rename, colour, delete, PIN), primary profile only

## 6. Verification

- [ ] 6.1 Desktop build without config: no gate, no Account section, v0.1 flows unchanged
- [ ] 6.2 Desktop build with a test config: sign in by scanning the QR code with a phone, restart and stay signed in, sign out
- [ ] 6.3 Create profiles on NuvioTV, sign in on the Switch and see them; set a PIN on the Switch and see the profile locked on NuvioTV
- [ ] 6.4 `ctest --test-dir build_desktop` passes (QR flow, refresh classification, profile logic)
- [ ] 6.5 Switch `.nro` builds; on hardware or in an emulator check that the QR code scans from a TV-docked screen and that the PIN pad works with the gamepad
