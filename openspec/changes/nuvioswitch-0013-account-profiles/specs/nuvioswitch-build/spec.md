## ADDED Requirements

### Requirement: Build-time service configuration
The build SHALL read service endpoints and client keys from an optional, gitignored `config.local.json` at the repository root, or from CMake cache variables of the same name, which take precedence. It SHALL generate `service_config.hpp` with one string constant per key. The recognised keys are:
- `NUVIO_SUPABASE_URL`, `NUVIO_SUPABASE_ANON_KEY`, `NUVIO_SUPABASE_FALLBACK_URL`
- `DEVICE_LOGIN_WEB_BASE_URL` (default `https://nuvio.tv/link`), `TV_LOGIN_WEB_BASE_URL` (default `https://nuvio.tv/tv-login`)
- `TRAKT_CLIENT_ID`, `TRAKT_CLIENT_SECRET`, `TRAKT_REDIRECT_URI`
- `SIMKL_CLIENT_ID`, `SIMKL_APP_NAME`
- `MDBLIST_CLIENT_ID`
- `TMDB_API_KEY`
- `PREMIUMIZE_CLIENT_ID`

A `config.example.json` listing every key with empty values SHALL be committed.

#### Scenario: No config file
- **WHEN** the project is configured without `config.local.json` and without `-D` overrides
- **THEN** the build succeeds, every key except the URL defaults is an empty string, and CMake prints which features are disabled

#### Scenario: CMake override
- **WHEN** CMake is run with `-DNUVIO_SUPABASE_URL=https://example.supabase.co` and `config.local.json` holds a different URL
- **THEN** the generated header contains `https://example.supabase.co`

#### Scenario: Secrets not committed
- **WHEN** a developer creates `config.local.json` with real keys
- **THEN** `git status` does not list it, and the generated header is only in the build directory
