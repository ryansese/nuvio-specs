# NuvioSwitch-0004 Home catalogs

## Why

After 0003 the app knows which add-ons are installed, but Home is still a placeholder. Browsing add-on catalogs is the first thing a user does in NuvioTV, and Search (0005) and Details (0006) build on the catalog request and the poster grid introduced here. This change makes Home a working add-on browser using NuvioTV's **Classic** layout.

## What Changes

- **Catalog requests** built exactly like NuvioTV's `CatalogRepositoryImpl.buildCatalogUrl`:
  - `{base}/catalog/{type}/{id}.json`, `{base}/catalog/{type}/{id}/skip={n}.json`, and `{base}/catalog/{type}/{id}/{k=v&...}.json`;
  - extra values are URL-encoded with `%20` for spaces, `skip` is added last, and the add-on's `?query` goes after `.json`;
  - the response's `metas` are parsed into catalog items: entries with no id or name are dropped, duplicate ids are removed, and the item type falls back to the catalog type.
- **Pagination** following `domain/model/CatalogDescriptorExtensions.kt`:
  - the skip step is `pageSize`, otherwise the smallest gap between numeric `skip` options, otherwise 100;
  - the next skip is `skip + itemsReturned`;
  - "has more" means the catalog supports `skip` and the last page wasn't empty.
- **Which catalogs appear on Home**:
  - every catalog of every enabled add-on, in add-on priority order and then manifest order;
  - except search-only catalogs (a required `search` extra) and catalogs with an explicit `showInHome: false` (`HomeViewModelCatalogUtils.shouldShowOnHome`).
- **Home tab in the Classic layout**:
  - a hero carousel of up to 7 items with artwork, spread across the loaded rows;
  - one horizontal row per catalog with the title "<Catalog name> - <Movie|Series|Type>" and a "from <add-on>" caption;
  - posters 126×189 with a title label;
  - more items load as the user scrolls near the end of a row.
  - The first 4 rows load immediately, and the rest load as they come near the screen (NuvioTV's `eagerCatalogLoadCount = 4`).
  - Rows that fail or come back empty are hidden. When no add-on provides catalogs, the message "No catalog add-ons installed" is shown.
- **"See all" screen** for a catalog: a paged poster grid that loads further pages as the user scrolls (`ui/screens/CatalogSeeAllScreen.kt`). It opens from the end of each row and from the row's options.
- **Selecting a poster** opens a temporary details stub showing the item's name, type and id, until Details (0006) lands.
- Home rebuilds when the add-on list changes: install, remove, reorder, enable or disable.

## Capabilities

### New Capabilities
- `nuvioswitch-home-catalogs`: the catalog request and pagination rules, which catalogs appear on Home and in what order, the Classic Home layout with hero and rows, lazy row loading, the See-all grid, and Home's empty and error states.

### Modified Capabilities
None. The Home placeholder from `nuvioswitch-app-shell` is replaced by real content, as the shell spec allows.

## Impact

- **Depends on:**
  - NuvioSwitch-0001: shell and Home tab;
  - NuvioSwitch-0002: `HttpClient`, the JSON helpers, `ImageLoader`, the response cache;
  - NuvioSwitch-0003: `AddonRepository`, `CatalogDescriptor` and `supportsExtra`, change subscription.
- **Code:** new `src/domain/meta_preview.hpp` and `catalog_row.hpp`, `src/data/catalog/` (URL builder, response parser, repository), `src/ui/home/` (tab, hero, row, poster cell) and `src/ui/catalog/` (See-all activity), plus XML layouts, i18n strings and unit tests.
- **Reused later:**
  - the catalog repository and poster cell by 0005 Search;
  - the poster-selection hook by 0006 Details, which replaces the stub;
  - Home's top area for the Continue Watching row added in 0009.
- **NuvioTV references:**
  - `data/repository/CatalogRepositoryImpl.kt`, `data/mapper/CatalogMapper.kt` and `domain/model/MetaPreview.kt`;
  - `domain/model/CatalogDescriptorExtensions.kt`;
  - `ui/screens/home/HomeViewModelCatalogUtils.kt` and `HomeViewModelCatalogPipeline.kt` (catalog selection, eager count, hero slots, display titles);
  - `ui/screens/home/ClassicHomeContent.kt`, `ui/components/HeroCarousel.kt`, `CatalogRowSection.kt` and `ContentCard.kt`;
  - `ui/screens/CatalogSeeAllScreen.kt`;
  - `ui/theme/SizeTokens.kt` for poster size.
