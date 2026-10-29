## ADDED Requirements

### Requirement: Saved library items
The app SHALL keep a per-profile library of saved items. Each item SHALL hold:
- `id`, `type`, `name`;
- `poster`, `posterShape`, `background`, `logo`;
- `description`, `releaseInfo`, `imdbRating`, `genres`;
- `addonBaseUrl`, `addedAt`.

An item SHALL be identified by `id` plus `type`. Adding an item that is already saved SHALL NOT create a duplicate.

#### Scenario: Add to library
- **WHEN** the user adds a film to the library from its Details screen
- **THEN** the film is saved with the current time as `addedAt` and appears in the Library tab

#### Scenario: No duplicates
- **WHEN** the user adds the same film again (for example from another device's restored data)
- **THEN** the library still holds one entry for it

### Requirement: Library tab
The Library tab SHALL show the saved items as a poster grid that can be navigated with the gamepad. Selecting an item SHALL open its Details screen. An empty library SHALL show a message explaining how to add items.

#### Scenario: Open from library
- **WHEN** the user selects a poster in the Library tab
- **THEN** the item's Details screen opens

#### Scenario: Empty library
- **WHEN** the library has no items
- **THEN** the tab shows "Your library is empty" with a hint to use "Add to library" on a title

### Requirement: Library sort options
The Library tab SHALL offer these sort options:
- provider order (the order items were stored in);
- date added, newest first or oldest first;
- title A–Z or Z–A.

The chosen option SHALL be remembered per profile.

#### Scenario: Sort by title
- **WHEN** the user picks "Title A–Z"
- **THEN** the grid is sorted by name, ignoring case, and the choice is still applied after restarting the app

### Requirement: Library type filter
The Library tab SHALL offer a type filter: All, Movies or Series.

#### Scenario: Series only
- **WHEN** the user picks "Series"
- **THEN** only items with type `series` are shown
