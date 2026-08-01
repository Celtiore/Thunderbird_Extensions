# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Monorepo containing three Thunderbird extensions (WebExtensions):

- **Email Cleaner** (v2.2+) — Manifest V3, auto-deletes emails based on sender/recipient/domain/subject/size filters with scheduling, dry-run simulation, and tri-state folder picker UI. Fork for Thunderbird 128–140+ compatibility.
- **Xpunge** (v5.0.2+) — Manifest V2, empties Trash/Junk and compacts folders across multiple accounts. Uses Experiment APIs (`browser.Xpunge.*`, `browser.LegacyPrefsMigrator.*`). Fork for Thunderbird 128–148 compatibility.
- **Contact Lens** (v1.1) — Manifest V3, scans all email accounts to extract, deduplicate, and classify contacts by relationship direction (bidirectional/received-only/sent-only). Features incremental scan, enriched export (subjects + body snippets), dashboard with tree view and folder picker, search, sort, CSV/JSON export. Paired with `/contact-analyze` skill for 4-pass analysis pipeline.

## Build & Packaging

No build system or dependencies — extensions are pure JavaScript. To package:

```bash
# Email Cleaner
cd email-cleaner/src && zip -r ../email-cleaner@example.com.xpi . -x '*.bak'

# Xpunge
cd xpunge/src && zip -r '../{786abda0-fd14-d247-bf69-38b2fc18491b}.xpi' .

# Contact Lens
cd contact-lens/src && zip -r ../contact-lens@example.com.xpi . -x '*.bak'
```

Install the `.xpi` in Thunderbird via Menu > Add-ons > Install from file.

Note: `.gitignore` excludes `**/src/` from git tracking — only `.xpi` files and docs are committed. Source files are present locally but untracked.

## Architecture

### Email Cleaner (`email-cleaner/src/`)
- `background.js` — monolithic script (~900 lines): preferences, context menus, batch message deletion logic, scheduling via `browser.alarms`, dry-run simulation. Uses Manifest V3 `browser.*` APIs.
- `options.js` / `options.html` — settings UI with tri-state folder tree picker, filter management, cleaning history table.
- `subject-filter-dialog.js` / `.html` — popup dialog for adding subject filters from context menu.
- `language-selector.js` — custom i18n system loading JSON from `web_accessible_resources/locales/`.
- `api/NotifyTools/` and `api/WindowListener/` — Experiment APIs for legacy/overlay communication.
- Locales: `_locales/{fr,en,de}/` (WebExtension i18n) + `web_accessible_resources/locales/{fr,en,de}.json` (custom UI strings).

### Xpunge (`xpunge/src/`)
- `background.js` — entry point (ES modules), delegates to `modules/`.
- `modules/xpunge.mjs` — core logic: empty trash/junk, compact, timer handling.
- `modules/menus.mjs` — tools menu and folder pane context menu entries.
- `modules/preferences.mjs` — preferences with defaults.
- `modules/migration.mjs` — migrates legacy prefs via `LegacyPrefsMigrator` Experiment API.
- `modules/folderPicker.mjs` — folder picker for options UI.
- `api/Xpunge/` — Experiment API (XPCOM): `emptyTrash`, `emptyJunk`, `compact`, `confirm` dialog.
- `api/LegacyPrefsMigrator/` — reads old-style Thunderbird prefs.
- Locales: `_locales/{en-US,fr-FR,da,de,el,it,ja,ru}/`.

### Contact Lens (`contact-lens/src/`)
- `background.js` — monolithic script: account/folder enumeration, batch message scanning, contact extraction/deduplication, incremental scan with per-folder timestamps, persistence via `browser.storage.local`, enriched export (subjects via `messages.query()`, body snippets via `messages.getFull()`), messaging to dashboard.
- `dashboard.html` / `dashboard.js` / `dashboard.css` — full-tab dashboard UI: tree view grouped by direction then account, folder picker with cascade selection, column sorting, text search, progress bar, enriched export overlay, CSV/JSON export.
- `options.html` / `options.js` — options page with data reset button and compatibility info.
- Locales: `_locales/{fr,en}/` (WebExtension i18n standard).

### Contact Analysis Pipeline (`exports/` + `.claude/skills/`)
- `exports/scans/` — dated enriched export scans (JSON + CSV) from Contact Lens.
- `exports/filters/exclusions.json` — domain/email/pattern exclusion rules (~287 domains, ~21 regex patterns).
- `exports/filters/keywords.json` — keyword categories for subject tagging (8 categories, FR+EN).
- `exports/results/` — output of `/contact-analyze` (enriched JSON, CSV, filter suggestions).
- `.claude/skills/contact-analyze.md` — Claude Code skill: 4-pass analysis (cleanup, Claude batch analysis, consolidation, filter validation).

## Key Conventions

- All user-facing strings use the WebExtension i18n system (`browser.i18n.getMessage()`), except Email Cleaner's options UI which uses a custom JSON-based locale system.
- Both extensions use `browser.storage.local` for preferences.
- Email Cleaner processes messages in batches (`DEFAULT_BATCH_SIZE = 100`) with delays to avoid UI freezes.
- Xpunge uses a 1-minute `browser.alarms` timer that checks preferences for scheduled runs.
- Contact Lens processes messages in read-only batches (`BATCH_SIZE = 100`) with 100ms delays, extracts contacts from `author`/`recipients` headers, and classifies relationship direction based on folder type (sent vs. other).
- XPI file names must match the extension IDs in their respective `manifest.json` (`email-cleaner@example.com`, `{786abda0-fd14-d247-bf69-38b2fc18491b}`, and `contact-lens@example.com`).
