# CLAUDE.md — Codebase Guide

## Project Overview

A personal dashboard PWA — a set of self-contained HTML apps sharing a common top/bottom navigation bar. No build step, no framework, no npm install required to run. Each page is a single HTML file with all CSS and JavaScript inline. Deployed on Vercel.

**Live pages:**
| File | Purpose |
|---|---|
| `index.html` | Goals tracker — Day Ring, Goal Ticker, To Do list |
| `health.html` | Supplement / daily stack tracker + WHOOP integration |
| `po-water.html` | Water intake tracker |
| `finance.html` | Net worth, subscriptions, wishlist, incoming orders |
| `gym.html` | Progressive overload gym tracker + progress photos |

---

## Repository Structure

```
/
├── index.html          # Goals / home page
├── health.html         # Health stack + WHOOP
├── po-water.html       # Water tracker
├── finance.html        # Finance tracker
├── gym.html            # Gym tracker
├── topbar.js           # Shared top bar + bottom tab bar (auto-injected)
├── sync.js             # Shared Supabase cloud-sync helper
├── manifest.json       # PWA manifest
├── icon-192.svg        # PWA icon
├── icon-512.svg        # PWA icon
├── package.json        # Only declares @supabase/supabase-js dep (for Vercel)
└── api/                # Vercel serverless functions
    ├── whoop-callback.js   # OAuth callback — exchanges code for tokens
    ├── whoop-refresh.js    # Refreshes WHOOP access token
    └── whoop-data.js       # Proxy for WHOOP API calls (v1 + v2)
```

---

## Architecture

### No Build Step
Every HTML file is standalone. Open in a browser directly — `file://` works for all pages except features that need the Vercel API routes (WHOOP OAuth). The CDN-loaded Supabase JS client (`cdn.jsdelivr.net`) is the only external runtime dependency.

### Shared Scripts
Two scripts are injected via `<script src="..." defer>` at the top of each page's `<body>`. Load order matters:

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<script src="sync.js" defer></script>
<script src="topbar.js" defer></script>
```

- **`topbar.js`** — Self-invokes on `DOMContentLoaded`. Injects the sticky top bar (water counter + finance link) and the bottom tab bar. Skips injection on `finance.html` and inside iframes. Reads `localStorage` to show water progress in the top bar.
- **`sync.js`** — Exposes `window.initCloudSync(config)`. Each page calls it once at startup to wire Supabase real-time sync for its `localStorage` keys.

**`gym.html`** has its own inline Supabase sync (does not use `sync.js`), and does NOT include `sync.js`.

### State — `localStorage` Only
All app state lives in browser `localStorage`. There is no user auth — the Supabase key is a publishable (anon) key. Supabase is used as a dumb KV store: one row per app in an `app_state` table, keyed by app name, with a JSON `data` column.

---

## localStorage Keys

| Key | Owner | Description |
|---|---|---|
| `po_water_v1` | `po-water.html`, `health.html`, `topbar.js` | Water intake logs + profile |
| `stack:items` | `health.html` | Supplement stack items |
| `stack:version` | `health.html` | Stack schema version |
| `stack:low` | `health.html` | Low-stock flags |
| `stack:taken:<date>` | `health.html` | Per-day taken state (prefix pattern) |
| `whoop_tokens_v1` | `health.html` | WHOOP OAuth tokens (access + refresh + expiry) |
| `po_coach_v1` | `gym.html` | Gym program definition + log |
| `po_coach_workout_done` | `gym.html` | Per-day workout completion |
| `po_coach_weights` | `gym.html` | Weight tracking history |
| `po_coach_photos` | `gym.html` | Progress photos (base64, compressed to ~1080px) |
| `subs` | `finance.html` | Subscription list |
| `wishlist` | `finance.html` | Wishlist items |
| `incoming_orders` | `finance.html` | Incoming order tracking |
| `nw_currency` | `finance.html` | Selected currency |
| `nw:activity` | `finance.html` | Net worth activity log |
| `nw:history` | `finance.html` | Net worth history snapshots |
| `finance_active_tab` | `finance.html` | Last active tab |
| Goals keys (various) | `index.html` | Goals tracker state |

---

## Cloud Sync (Supabase)

**Supabase project:** `vrtbwgizjohfuhmoikoj.supabase.co`  
**Key:** `sb_publishable_l2Pf9FqV_s-68wIU-HMhfw_ajnjmqUk` (publishable/anon — safe to commit)

### `sync.js` — `initCloudSync(config)`

Each page calls this once. Config shape:

```js
initCloudSync({
  appKey: 'goals',               // Supabase row key in app_state table
  syncedKeys: ['key1', 'key2'],  // Exact localStorage keys to sync
  syncedPrefixes: ['prefix:'],   // localStorage key prefixes to sync
  onApplied: () => { /* re-render after remote data applied */ },
});
```

Sync flow:
1. On init — fetches remote state from Supabase, applies to `localStorage` if newer.
2. Patches `localStorage.setItem`/`removeItem` — debounces pushes by 250ms after any write to a synced key.
3. Subscribes to Postgres real-time changes — applies remote updates immediately.
4. On `beforeunload`/`pagehide` — fires a synchronous `fetch` with `keepalive: true` to flush any pending state.

### Supabase App Keys

| App | `appKey` | Synced via |
|---|---|---|
| Goals / index | `goals` | `sync.js` |
| Health / water | `health` | `sync.js` |
| Water (standalone) | `health` | `sync.js` (same row as health) |
| Finance | `finance` | `sync.js` |
| Gym | `po-coach` | Inline sync in `gym.html` |

---

## WHOOP Integration

WHOOP is wired only into `health.html`. OAuth flow:

1. User clicks "Connect WHOOP" → redirected to `https://api.prod.whoop.com/oauth/oauth2/auth`
2. WHOOP redirects to `/api/whoop-callback` (Vercel function)
3. Callback exchanges code for tokens, redirects to `health.html#whoop_access=...&whoop_refresh=...&whoop_expires=...`
4. `health.html` reads the hash, stores tokens in `localStorage` under `whoop_tokens_v1`, clears the hash.

### API routes (all in `api/`)

| Route | Method | Purpose |
|---|---|---|
| `/api/whoop-callback` | GET | OAuth code → token exchange, redirects to `health.html` |
| `/api/whoop-refresh` | POST `{ refresh_token }` | Returns new access token JSON |
| `/api/whoop-data` | GET `?path=/v2/...` | Proxy to WHOOP API (passes `Authorization` header through) |

**WHOOP API versioning:** `whoop-data.js` uses v1 for `/cycle` paths, v2 for everything else.

**Required Vercel environment variables:**
- `WHOOP_CLIENT_ID`
- `WHOOP_CLIENT_SECRET`
- `WHOOP_REDIRECT_URI` — hardcoded to `https://row-pink.vercel.app/api/whoop-callback`

---

## `topbar.js` Conventions

- Skips rendering on `finance.html` (detected by pathname) and inside iframes.
- Bottom tab bar tabs: `main` (`index.html`), `health` (`health.html`), `fitness` (`gym.html`).
- Water button in top bar increments `po_water_v1.logs[<YYYY-MM-DD>]` and triggers a Supabase push (merged into the `health` row, only when not on `health.html`).
- Dot color: `#7DD3FC` (good) → `#fbbf24` (warn, ≥50% of goal) → `#ff8a8a` pulsing (miss, <50% after 6pm).
- Locks gesture events on mobile (prevents pinch-zoom). Adds `body.topbar-modal-open` when modals are open.

---

## Visual / Style Conventions

All pages share the same design language:

- **Dark background:** `#050506` page, `#0a0a0b` chrome.
- **Font stack:** `-apple-system, BlinkMacSystemFont, "Inter", "Segoe UI", Roboto, sans-serif`. Monospace: `ui-monospace, "SF Mono", Menlo, Consolas, monospace`.
- **CSS variables:** `--text-primary: #FAFAFA`, `--text-secondary: #B8B6B0`, `--text-tertiary: #76746E`, `--success: #6BE3A4`, `--warning: #F2C063`, `--danger: #FF6B6B`.
- **Card chassis:** `rgba(255,255,255,0.04)` bg, 16px radius, `backdrop-filter: blur(24px) saturate(1.2)`.
- **Centered layout:** max-width ~480–600px on mobile, wider for desktop tabs.
- **No visible scrollbar on mobile** — hidden via `::-webkit-scrollbar` + `scrollbar-width: none`.
- **Safe-area aware:** All padding uses `env(safe-area-inset-*)` for iPhone notch/Dynamic Island.

---

## Development Workflow

### Running Locally
No build step — open any `.html` file directly in a browser. For WHOOP OAuth you need a live Vercel deploy (localhost redirect URIs won't match the hardcoded `WHOOP_REDIRECT_URI`).

### Deploying
Push to `main` → Vercel auto-deploys. The Vercel project must have `WHOOP_CLIENT_ID`, `WHOOP_CLIENT_SECRET`, and `WHOOP_REDIRECT_URI` set as environment variables.

### Making Changes

- **Adding a feature to one page:** Edit that page's HTML file. All JS and CSS are inline — find the relevant `<style>` block or `<script>` block.
- **Adding a shared nav item:** Edit `topbar.js` — update `bottombarHtml`, `currentPageKey()`, and the CSS.
- **Adding a new synced localStorage key:** Add it to the page's `initCloudSync({ syncedKeys: [...] })` call.
- **Adding a new API route:** Create a new file in `api/` exporting a default `handler(req, res)` function (Vercel serverless style).

### Git Branch
Active development branch: `claude/claude-md-docs-ObWZu`

---

## Key Invariants

1. **Each HTML file is self-contained.** Do not create shared CSS files or shared JS modules — the zero-build-step constraint is a feature.
2. **All state is in `localStorage`.** Never add a login flow or server-side user state.
3. **Supabase key is publishable.** It's safe in source. Do not use secret keys in frontend code.
4. **`finance.html` gets no topbar/bottombar chrome** — `topbar.js` detects and skips it by pathname.
5. **`gym.html` does not use `sync.js`** — it has its own inline Supabase sync block.
6. **WHOOP tokens are not synced to Supabase** — they stay in `localStorage` only (not in `syncedKeys`).
7. **Progress photos are compressed to ~1080px before storing** to avoid exceeding `localStorage` quota (noted in a code comment in `gym.html`).
