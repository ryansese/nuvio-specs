## Purpose

TBD: Define the NuvioSwitch HTTP client: default headers, timeouts, TLS modes, the response cache, threading and how errors are reported.

## Requirements

### Requirement: Asynchronous HTTP requests
The app SHALL perform HTTP requests off the UI thread on a worker pool and deliver each result exactly once on the UI thread. A result SHALL be either a success (status 2xx with body, headers and final URL) or an error (HTTP status with body, timeout, connection failure, TLS failure, or cancelled).

#### Scenario: Successful GET
- **WHEN** app code requests `GET https://v3-cinemeta.strem.io/manifest.json`
- **THEN** the UI stays responsive while the request runs, and the callback runs on the UI thread with status 200 and the response body

#### Scenario: HTTP error
- **WHEN** the server answers 404
- **THEN** the callback receives an error carrying status 404 and the body, not a success

#### Scenario: Parallel requests
- **WHEN** ten requests are started at once
- **THEN** up to the pool size (4) run concurrently and the rest queue, and every request gets exactly one callback

#### Scenario: Cancelled request
- **WHEN** app code cancels a request that hasn't completed
- **THEN** its success or error callback is not invoked with data, and no UI code runs for it

### Requirement: Default request headers and timeouts
Every request SHALL send `User-Agent: NuvioSwitch/<version>` and an `Accept-Language` header built from the app locale as `<locale>,en;q=0.7`, or just `<locale>` when the locale is `en` or `en-*` (and `en` when the locale is unknown), matching `buildAcceptLanguageHeader` in NuvioTV's `core/di/NetworkModule.kt`. Per-request headers SHALL be added on top and override defaults with the same name. Connect timeout SHALL be 30 s and the read (low-speed) timeout 60 s unless the caller sets others. Redirects SHALL be followed up to 10 hops.

#### Scenario: Headers sent
- **WHEN** a request is made with locale `it-IT` and app version `0.1.0`
- **THEN** the server receives `User-Agent: NuvioSwitch/0.1.0` and `Accept-Language: it-IT,en;q=0.7`

#### Scenario: Server stalls
- **WHEN** a server accepts the connection but sends nothing for 60 seconds
- **THEN** the request fails with a timeout error

### Requirement: Permissive TLS for add-on requests
Requests marked as add-on requests SHALL skip certificate and hostname verification, as NuvioTV's `addonPermissive` client does, so self-hosted add-ons with self-signed certificates work. All other requests SHALL verify TLS.

#### Scenario: Self-signed add-on
- **WHEN** an add-on request goes to an HTTPS server with a self-signed certificate
- **THEN** the request succeeds

#### Scenario: Non-add-on request with bad certificate
- **WHEN** a normal request goes to a server with an invalid certificate
- **THEN** it fails with a TLS error

### Requirement: HTTP response cache
GET responses SHALL be cached on disk, up to 50 MB in total, with least-recently-used entries evicted first. A response SHALL be reused without a network call while it is fresh according to `Cache-Control: max-age` (or `s-maxage`). Responses with `no-store`, `no-cache`, `private`, or a status outside 2xx SHALL NOT be stored. Callers SHALL be able to bypass the cache per request.

#### Scenario: Fresh cached response
- **WHEN** a URL returned `Cache-Control: max-age=600` and is requested again 5 minutes later
- **THEN** the cached body is returned without a network request

#### Scenario: Expired response
- **WHEN** the same URL is requested 11 minutes later
- **THEN** a network request is made and its response replaces the cache entry

#### Scenario: Error not cached
- **WHEN** a URL returns 500
- **THEN** nothing is stored, and the next request for it goes to the network

#### Scenario: Offline fallback
- **WHEN** the network request fails with a connection error and an expired cached response exists
- **THEN** the caller can opt in to receive the stale cached body, flagged as stale
