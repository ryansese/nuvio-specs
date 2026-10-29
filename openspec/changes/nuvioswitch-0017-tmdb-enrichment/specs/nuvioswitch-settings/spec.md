## ADDED Requirements

### Requirement: TMDB section
When `TMDB_API_KEY` is set, Settings SHALL have a per-profile TMDB section with these defaults, matching `data/local/TmdbSettingsDataStore.kt`:
- Enable TMDB enrichment: off;
- Language: `en`, chosen from a list of ISO 639-1 codes with optional region;
- Artwork, Basic info, Details, Credits and Episodes: all on.

The group toggles are disabled while enrichment is off.

#### Scenario: Turn on in Italian
- **WHEN** the user enables TMDB enrichment and picks `it-IT`
- **THEN** the next details screen shows Italian titles and descriptions where TMDB has them
