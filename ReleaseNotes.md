# CMAP — Release Notes

## v1.0.RC2 — 2026-10-02

### Design
- Complete visual redesign to match the Investor design language: Fraunces headings, Inter body text and IBM Plex Mono data, warm cream / coffee-black themes with coral and teal accents.
- Top bar: logo, name and version pill on the left; database pill, round icon buttons, sliding theme switch and menu on the right. Release Notes moved into the menu.
- Map / Items / Cards tabs with serif labels and a coral underline, directly under the top bar.
- Cards live in a labelled panel with a card count, a search box, an inline "Hide disabled cards" checkbox and a compact data table with sticky headers.
- Edit, Del, Purge and Recover are now small icon buttons; Recover is highlighted in teal, Purge in coral.
- Restyled Card dialog, confirmation dialogs, toasts (bottom-right), type-ahead and icon picker popovers, and the footer.

## v1.0.RC1 — 2026-10-02

### New Features
- Database management: create a new database (default name `CMAP.sqlite`) or open an existing one. The current file name, save status, Save and Switch Database controls appear in the header.
- Automatic saving to the opened file after every change (Chrome / Edge, File System Access API), plus a manual Save button.
- Reopen the last database with one click from the start screen.
- Fallback for other browsers: open with a file picker and save by downloading; the app tells you when auto-save isn't available.
- Map, Items and Cards tabs. Map and Items show a "coming soon" state.
- Cards tab: filter by name or title, "Hide disabled cards" switch, New Card button, and a table with properties and record counts.
- New / Edit Card dialog: Card Name with live uniqueness check (spaces and letter case are ignored), Title auto-filled from the name, Description, Template, and an editable Properties table (type, label, key, visibility, order).
- Property types: Icon, Text, Number, Enum, Date.
- Disable (Del) and Recover cards; Purge permanently deletes a card with all its properties, values and Items. Every destructive step asks for confirmation.
- Dark / light theme toggle, remembered between visits.

### Architecture
- sql.js 1.13.0 (SQLite in WebAssembly), loaded from cdnjs. The first launch needs an internet connection; the app shows a clear message if the engine can't load. This is a deliberate exception to the "no CDN" rule for CMAP.
- All data access goes through a single adapter block, ready to be swapped for Supabase later.
- Portable schema in a single block: TEXT UUID ids, ISO 8601 timestamps, CHECK constraints, no AUTOINCREMENT, PRAGMA or STRICT.
- Foreign-key cascades are switched on at connection time (SQLite runtime setting, outside the schema block).
- `meta` table with `app` and `schema_version`; non-CMAP or newer-format files are rejected with a clear message.
- Enum type-ahead and Icon picker components (186 embedded Tabler icons, MIT) are built and tested, ready for the Items phase.
