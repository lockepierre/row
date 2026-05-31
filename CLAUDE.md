# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the project

No build step, no install required. Open any `.html` file directly in a browser or use VS Code Live Server. For the Vercel serverless API routes (`/api/*.js`), deploy to Vercel or run locally with `vercel dev`.

## Architecture

This is a **no-build, no-framework personal dashboard PWA** — all styles and scripts are inline inside each HTML file. There is no bundler, no transpilation, and no component library.

### Page structure

Each `.html` file is a fully self-contained app. CSS lives in a single `<style>` block; JavaScript lives in one `<script>` block at the bottom. Pages share nothing except two injected scripts:

- **`topbar.js`** — self-injects the sticky top bar (water counter + finance link) and the bottom tab nav into every page that includes `<script src="topbar.js" defer>`. It is skipped on `finance.html` and inside iframes (so `po-water.html` can embed cleanly). It also handles mobile gesture locking and modal scroll locking.
- **`sync.js`** — exposes `window.initCloudSync({appKey, syncedKeys, syncedPrefixes, onApplied})`. Pages call this after loading Supabase to get bidirectional real-time cloud sync of their localStorage state. Patches `localStorage.setItem`/`removeItem` to schedule pushes; pulls on page load and subscribes to Postgres realtime changes.

Both scripts require `<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2">` loaded before them.

### Pages

| File | Purpose |
|---|---|
| `index.html` | Goals tracker — Day Ring, Goal Ticker, Today/Tomorrow To Do lists |
| `health.html` | Supplement stack tracker + WHOOP integration |
| `gym.html` | Progressive overload gym tracker with configurable split rotation |
| `finance.html` | Finances (no topbar/bottombar) |
| `po-water.html` | Water intake tracker (also embeddable as iframe) |

### State / storage

All app state lives in `localStorage`. Common key shapes:
- `goals:YYYY-MM-DD` → `[{ text, done, doneAt?, queued? }]`
- `goal_streak_v1` → `{ count, lastProcessedDate }`
- `po_water_v1` → `{ unit, bottleMl, glassMl, weightUnit, profile, caffeineMgPerDay, substances, logs }`

**6 AM day boundary**: `index.html` treats 6 AM as the start of the new day (not midnight). `getActiveDateString()` subtracts a day if `getHours() < 6`. `getTomorrowDateString()` follows the same logic. When adding date logic, always account for this.

### Serverless API (`/api/`)

Three Vercel serverless functions handle the WHOOP OAuth flow. They require environment variables: `WHOOP_CLIENT_ID`, `WHOOP_CLIENT_SECRET`, `WHOOP_REDIRECT_URI`.

- `whoop-callback.js` — receives OAuth code, exchanges for tokens, redirects to `health.html#whoop_access=...&whoop_refresh=...&whoop_expires=...`
- `whoop-data.js` — CORS proxy to the WHOOP API (avoids direct browser-to-WHOOP CORS issues). Routes `/cycle*` paths to API v1, all others to v2.
- `whoop-refresh.js` — refreshes an expired WHOOP access token using the stored refresh token.

### Supabase

The project URL and publishable key are hardcoded in `topbar.js` and `sync.js` (they are intentionally public/publishable, not secret). Each page syncs its state under a different `appKey` in the `app_state` table (columns: `key`, `data` JSONB, `updated_at`).

### External API calls from the browser

`index.html` calls the Anthropic API directly from the browser when the "Polish" button is used. The key is declared as `const ANTHROPIC_API_KEY = '';` at the top of the script block — if empty, polish falls back to plain add. The request header `anthropic-dangerous-direct-browser-access: true` is required.

### PWA

`manifest.json` configures standalone display mode. `topbar.js` sets `body.has-bottombar` which adds bottom padding so content doesn't hide behind the fixed nav.

## Key conventions

- **Design system**: dark theme, `#050506` background, glass cards (`rgba(255,255,255,0.04)` + `backdrop-filter: blur`), film-grain `body::after`, animated radial gradient `body::before`. CSS variables in `:root`: `--text-primary`, `--text-secondary`, `--text-tertiary`, `--success`, `--warning`, `--danger`, `--font`, `--font-mono`.
- **Mono font** is used for all numbers, times, and data values.
- **`finance.html`** deliberately has no topbar/bottombar — preserve this.
- **`gym.html`** has a `CONFIG` block at the top of the script for user customization (gyms, training days, split rotation, progression rules). Keep user-facing config here, not mixed into app logic.
- When a page fires a custom event after mutating localStorage (e.g. `goals-changed`), other components on the same page listen for it to stay in sync without waiting for the next polling interval.
