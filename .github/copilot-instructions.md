# MMM-PublicTransportHafas AI Coding Instructions

## Project Overview

This is a **MagicMirror² module** that displays public transport departures from HAFAS-based transit APIs. It uses a dual-architecture client-server pattern with the MagicMirror framework's module lifecycle.

### Key Architecture Components

1. **Frontend Module** (`MMM-PublicTransportHafas.js`): Browser-side MagicMirror module
   - Extends `Module` class from MagicMirror framework
   - Must implement: `getDom()`, `getStyles()`, `getScripts()`, `getTranslations()`
   - Communicates via `socketNotificationReceived()` / `sendSocketNotification()`
   - Uses global variable `UserPresence` for PIR sensor integration (must use `var`)

2. **Backend Helper** (`node_helper.js`): Node.js server-side component
   - Extends `NodeHelper` class
   - Handles API calls to HAFAS/Vendo clients (db-vendo-client for `db`/`dbweb`, hafas-client for other profiles)
   - Manages fetcher instances per station ID in `departuresFetchers` array

3. **Core Services**:
   - `DepartureFetcher.mjs`: ESM module handling API communication, departure filtering, and reachability calculation
   - `PtDomBuilder.js`: Creates wrapper and table structure
   - `PtTableBodyBuilder.js`: Builds table rows with departure data, delay coloring, warnings, and marquee effects

### Data Flow

```text
Frontend Module → sendSocketNotification("CREATE_FETCHER") →
Node Helper creates DepartureFetcher →
sends "FETCHER_INITIALIZED" →
Frontend starts interval loop →
sends "FETCH_DEPARTURES" →
DepartureFetcher queries API →
"DEPARTURES_FETCHED" or "FETCH_ERROR" →
Frontend updates DOM via getDom()
```

## Development Conventions

### File Types and Patterns

- **ESM modules**: `*.mjs` files (DepartureFetcher, query_stations utility)
- **CommonJS**: `*.js` files (main module, node_helper, DOM builders)
- **Import paths**: Node helpers use relative imports (`require("../../js/logger")`); ESM uses `.js` extensions
- **Global dependencies**: dayjs plugins loaded via script tags, accessed as `window.dayjs_plugin_*`

### MagicMirror Module Lifecycle

Never directly manipulate DOM outside `getDom()`. MagicMirror calls `getDom()` on every `updateDom()` trigger.

**Critical methods**:

- `start()`: Initialize state, create fetchers, DO NOT start intervals here
- `socketNotificationReceived("FETCHER_INITIALIZED")`: Start fetch loop only after backend confirms ready
- `getDom()`: Return fresh DOM wrapper on every call (stateless)
- `notificationReceived("USER_PRESENCE")`: Pause/resume updates based on PIR sensor
- `notificationReceived("MODULE_DOM_CREATED")`: Module visibility changed (handles carousel/pages integration)

### Configuration Handling

- All config sanitization in `sanitizeConfig()` method (e.g., minimum `updatesEvery` is 30s)
- `timeInFuture` auto-adjusted to `timeToStation + 30` minimum
- Station/line filtering happens in `DepartureFetcher`, not DOM builders

### Error Handling Pattern

Uses threshold-based error tolerance via `discardSocketErrorThreshold`:

- Errors below threshold: Keep showing last valid data, display warning in "Last update" line
- Errors above threshold: Clear departures, show error message
- Socket errors log as warnings until threshold exceeded, then as errors

### CSS and Styling

- Base styles: `css/styles.css`
- Line-specific colors: `css/{city}-lines.css` (e.g., `leipzig-lines.css`)
- CSS classes derived from line names: `"STR 11"` → `.str11` (lowercase, no spaces)
- Color classes: `.mmm-pth-has-delay` (red), `.mmm-pth-to-early` (green)
- Marquee: Direction names >24 chars scroll if `marqueeLongDirections: true`

## Testing & Quality

**Run before commits**:

```bash
node --run test          # Lint + spelling
node --run lint:fix      # Auto-fix linting/formatting
node --run test:spelling # Check spelling only
```

**Key tools**:

- ESLint with flat config (`eslint.config.mjs`) using `@eslint/js`, `@stylistic`, `eslint-plugin-import-x`
- Prettier (disabled embedded language formatting via `prettier.config.mjs`)
- cSpell for spell checking (custom words in `cspell.config.json`)
- Husky + lint-staged for pre-commit hooks

**ESLint overrides**:

- `max-statements: 25`, `max-lines-per-function: 100`, `max-params: 4`
- Ignore unresolved: `"eslint/config"`, `"node_helper"`, `"logger"` (MagicMirror globals)

## Common Tasks

### Adding a New Configuration Option

1. Add to `defaults` object in `MMM-PublicTransportHafas.js`
2. Add validation to `sanitizeConfig()` if needed
3. Pass to fetcher in `fetcherOptions` if backend needs it
4. Update README.md configuration table

### Adding a New HAFAS Profile

Profiles auto-load from `hafas-client/p/{profile}/` or `db-vendo-client/p/{profile}/`. Check if `db`/`dbweb` for Vendo client, else use HAFAS client. See `DepartureFetcher.init()`.

### Station Query Utility

```bash
node --run query [profile]  # Default: db
```

Finds station IDs for locations. Lives in `convenience/query_stations.mjs`.

### DOM Updates and Delays

- **Delay display**: When `showAbsoluteTime: true`, delay shown as `+5` (minutes late) or `+?` (no real-time data)
- **Relative time**: When `showAbsoluteTime: false`, shown as "in 5 minutes" (localized via dayjs)
- **Table headers**: Can be symbols (icons) or text based on `showTableHeadersAsSymbols`

## Critical Gotchas

- **Direction filtering broken in `db` profile** (issue [#193](https://github.com/KristjanESPERANTO/MMM-PublicTransportHafas/issues/193)): Use `dbweb` or regional profiles instead
- **Fetcher initialization**: Node helper sends `FETCHER_INITIALIZED` before async init completes to prevent UI blocking. Actual HAFAS client init happens in background.
- **Multiple instances**: Each module instance identified by `this.identifier` (auto-generated by MagicMirror)
- **dayjs locale**: Auto-loaded based on `config.language` in `getScripts()`
- **Translation keys**: Use format `PTH_*` for public transport specifics, generic messages use base keys

## File Structure Reference

- **Frontend logic**: `MMM-PublicTransportHafas.js` + `core/PtDomBuilder.js` + `core/PtTableBodyBuilder.js`
- **Backend logic**: `node_helper.js` + `core/DepartureFetcher.mjs`
- **Translations**: `translations/{en,de}.json`
- **Line styles**: `css/{city}-lines.css` (Leipzig, Berlin, Hamburg, etc.)
- **Utilities**: `convenience/query_stations.mjs`

## Version Requirements

- Node.js ≥20 (see `package.json` engines)
- MagicMirror² ≥2.31.0 (see `requiresVersion` in module)
