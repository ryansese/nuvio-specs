## Context

NuvioTV signs in to the shared Nuvio Supabase backend with a device-login flow: the TV shows a QR code, a phone approves, and the TV exchanges a one-time code for tokens (`core/auth/AuthManager.kt` lines 58–64 and 550–760, `ui/screens/account/AccountViewModel.kt` 273–400). NuvioTV uses supabase-kt, but every call it makes is plain HTTP: PostgREST RPCs under `/rest/v1/rpc/*`, GoTrue under `/auth/v1/*` and one Edge Function. That makes it a direct fit for the libcurl client from 0002.

Profiles are local first (`core/profile/ProfileManager.kt`, `profiles_json` in the global profile store) and synced through `sync_push_profiles` / `sync_pull_profiles`. PINs live only on the server; `ProfileLockStateDataStore` caches just which profiles are locked.

From 0002, the app has per-profile stores at `profiles/<id>/*.json` and global stores at the data root, with profile 1 fixed until now.

## Goals / Non-Goals

**Goals:**
- Sign in on the Switch without typing an email or password.
- Keep the session across launches and survive token expiry.
- Profiles that behave like NuvioTV's, so an account set up on the TV looks the same on the Switch.
- Leave a clean "Continue without account" path, and a build with no secrets that still works.

**Non-Goals:**
- Email/password sign-in and sign-up (NuvioTV has them, but they're awkward on a gamepad; can be added later).
- Syncing add-ons, library, progress and watched items (0014).
- Avatar images and profile backgrounds (the colour and initial are drawn instead), and "copy settings from profile".
- Custom server configuration (`data/local/ServerConfigurationStore.kt`) and legacy sync codes (`generate_sync_code`/`claim_sync_code`).

## Decisions

**1. Config is generated at build time from JSON.**
- CMake reads `config.local.json` with `file(READ)` + `string(JSON ...)` (CMake ≥ 3.19), lets `-D` cache variables override it, and writes `service_config.hpp` with `configure_file`.
- Alternative: read a config file from the SD card at runtime. Rejected for the main keys: NuvioTV bakes them in, and asking users to supply backend keys isn't viable. A runtime override can come later with custom servers.

**2. A thin Supabase client in `src/core/account/supabase_client.*`, not a library.**
- It offers `rpc(name, json)`, `get(table, filters)`, `functionCall(name, json)` and the auth token endpoints.
- It adds `apikey`/`Authorization` headers and the single expired-JWT refresh-and-retry, like `withJwtRefreshRetry` in NuvioTV.
- The refresh is serialised with a mutex, as in `refreshCurrentSessionSerialized`.
- What counts as an invalid refresh token follows `isInvalidAuthRefreshResponse`: an auth error with messages such as "invalid refresh token" or "refresh_token_not_found". 408/5xx/52x responses count as transient.

**3. The QR flow is a pure state machine** (`QrLoginFlow`) with the states Idle → Starting → Showing(code, url, expiresAt, interval) → Exchanging → SignedIn, plus Expired and Failed.
- HTTP calls and timers are injected, so the whole flow can be tested with ctest on desktop using canned responses: approved, expired, used, cancelled, the legacy fallback, and a slow poll.
- The poll interval is `max(2, poll_interval_seconds)`, matching `AccountViewModel.kt:355`.

**4. QR rendering.**
- Vendor `nayuki/QR-Code-generator` (C++, MIT, two files) under `src/thirdparty/qrcodegen`.
- Draw the modules as a nanovg-filled `brls::View` subclass (`QrCodeView`), so no image encoding is needed.
- Use error-correction level M with a 1-module margin, as `QrCodeGenerator.generate(..., margin = 1)` does.

**5. Device identity.**
- `p_device_name` is "Nintendo Switch" (or "NuvioSwitch (macOS)" on desktop).
- `p_device_type` defaults to `"tv"`, the value the backend already accepts. Making it a `"switch"` constant is the open question below.
- The nonce uses libnx `randomGet` on Switch and `std::random_device` on desktop.

**6. Session storage.**
- A global `auth_session` store holds `access_token`, `refresh_token`, `expires_at`, `user_id` and `email`. A global `onboarding` store holds `has_seen_auth_qr`.
- Tokens are stored in plain JSON on the SD card. The Switch has no keystore available to homebrew; this matches how other homebrew apps store tokens. Documented in the README.

**7. Profiles.**
- `ProfileManager` owns the global `profiles` store (`profiles`, `active_profile_id`, `pin_enabled_map`).
- Switching profiles notifies services through a simple observer list, so per-profile repositories reload.
- Remote fields the Switch doesn't use are kept in an `extra` JSON object per profile and sent back unchanged.

**8. Launch order** follows `MainActivity.kt`:
1. account gate (first launch, config present, not signed in, not skipped);
2. profile pull (signed in);
3. profile selection, when there is more than one profile or a lock;
4. main shell.

The gate and the selection screen are activities pushed before `MainActivity`.

## Risks / Trade-offs

- [The backend may reject or mislabel `p_device_type: "switch"`] → Ship with `"tv"`, and flip the constant once the backend owner confirms.
- [The anon key and client IDs end up in a public `.nro`] → Same exposure as NuvioTV's APK. Public CI builds leave them empty; only release builds get them.
- [Tokens in plain text on the SD card] → Accepted and documented. Sign out deletes them.
- [The QR code may be too dense to scan on a TV screen] → Use the short `verification_uri_complete`, keep module size at least 6 px, and show `user_code` plus `verification_uri` as a fallback.
- [`string(JSON)` needs CMake 3.19] → devkitPro and Homebrew both ship newer; set `cmake_minimum_required` accordingly for this feature.

## Open Questions

- Should `p_device_type` be `"switch"`? This needs confirmation from the Nuvio backend owner: do `start_device_login_session` and `linked_devices` accept new types, and how does the web approval page label them? Until then, `"tv"`.
- Should the Switch build use the production Nuvio backend, or a separate project, while the port is unofficial?
