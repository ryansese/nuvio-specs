# NuvioSwitch-0013 Account QR sign-in and profiles

## Why

After v0.1, NuvioSwitch works offline only. NuvioTV, NuvioMobile and NuvioDesktop share one Nuvio account (Supabase), so a user's add-ons, library and progress follow them between devices. To join that, the Switch first needs to sign in, which is hard with a gamepad and an on-screen keyboard. The NuvioTV QR device-login flow fixes this: scan a code with a phone, approve, done. Profiles, with an optional PIN, are part of the same account model, and sync (0014) is scoped per profile, so both land together.

## What Changes

- **Build-time service config.** A gitignored `config.local.json` (or CMake `-D` values) holds the backend endpoints and keys. It's baked into a generated `service_config.hpp`. Keys mirror NuvioTV's BuildConfig: `NUVIO_SUPABASE_URL`, `NUVIO_SUPABASE_ANON_KEY`, `NUVIO_SUPABASE_FALLBACK_URL`, `DEVICE_LOGIN_WEB_BASE_URL`, `TV_LOGIN_WEB_BASE_URL`. The integration keys used by later changes (`TRAKT_CLIENT_ID`, `TRAKT_CLIENT_SECRET`, `SIMKL_CLIENT_ID`, `MDBLIST_CLIENT_ID`, `TMDB_API_KEY`, `PREMIUMIZE_CLIENT_ID`) are reserved now. Empty values switch the matching feature off.
- **QR sign-in.** Ported from `core/auth/AuthManager.kt` and `ui/screens/account/AccountViewModel.kt`:
  - `rpc/start_device_login_session`, falling back to `rpc/start_tv_login_session`;
  - poll `rpc/poll_tv_login_session`;
  - exchange via `functions/v1/tv-logins-exchange`;
  - the QR code and the short code are shown with a countdown.
- **Session handling.** Tokens are stored, refreshed through `/auth/v1/token?grant_type=refresh_token`, and refreshed again once when a request fails because the JWT expired. The effective user ID comes from `rpc/get_sync_owner`. The device is registered with `rpc/register_current_device`. Sign out clears the session.
- **First-launch gate.** On first launch, a sign-in screen offers **Continue without account**. That choice is remembered and keeps the app fully local, as NuvioTV's `MainActivity.kt` does.
- **Profiles.** Up to 6 profiles (`ProfileManager.MAX_PROFILES`). Profile 1 is primary. Each has a name, an avatar colour, and "use primary add-ons". A "Who's watching?" screen appears when there is more than one profile or the active profile has a PIN.
- **PIN lock.** A 4-digit PIN per profile, through `set_profile_pin`, `clear_profile_pin`, `verify_profile_pin` and `sync_pull_profile_locks`. The PIN is checked by the server, so PINs need a signed-in account.
- **Profile sync.** The profile list itself is pushed and pulled with `sync_push_profiles` / `sync_pull_profiles`. Removed profiles' data is deleted with `sync_delete_profile_data`.
- **Settings.** Account and Profiles sections are added to the Settings screen. They're shown only on the primary profile, as in NuvioTV.

## Capabilities

### New Capabilities
- `nuvioswitch-account`: build-time service config, QR device sign-in, session storage and refresh, effective user and device registration, sign out, and the first-launch account gate.
- `nuvioswitch-profiles`: local profiles, profile selection, PIN lock, and profile list sync.

### Modified Capabilities
None. Requirements for capabilities owned by earlier changes are added as `ADDED` delta specs, merged after those changes are archived:
- `nuvioswitch-build`: generated service config header and the gitignored `config.local.json`.
- `nuvioswitch-settings`: Account and Profiles sections.

## Impact

- **Depends on:**
  - 0001 (build, app shell)
  - 0002 (HTTP client, per-profile stores: `profiles/<id>/` already exists with profile 1 as default)
  - 0011 (Settings screen)
  - 0012 (release packaging, so a config-less public build still works)
- **Code:** new `src/core/account/` (Supabase client, auth session, QR flow state machine), `src/core/profile/`, `src/activity/` screens (sign-in, profile selection, PIN pad, profile editor), CMake config generation, and a QR encoder (vendored single-file library).
- **Backend:** uses the existing Nuvio Supabase RPCs with no schema changes. Whether `p_device_type` can be `"switch"` instead of `"tv"` needs backend confirmation (see design.md).
- **Security:** the anon key and other client IDs are compiled into the `.nro`, as in NuvioTV's APK. Real values are never committed; CI builds without them, and those builds run local-only.
- **NuvioTV references:** `core/auth/AuthManager.kt`, `core/auth/DeviceSessionRegistration.kt`, `ui/screens/account/AccountViewModel.kt`, `ui/screens/account/AuthQrSignInScreen.kt`, `core/profile/ProfileManager.kt`, `core/sync/ProfileSyncService.kt`, `ui/screens/profile/ProfileSelectionScreen.kt`, `data/local/ProfileLockStateDataStore.kt`, `MainActivity.kt` (startup gating).
