**Project:** marathon-hub-race-scraper

**Platform:** running.moximoxi.net — Japanese marathon platform for Chinese runners

**GitHub:** [https://github.com/jason-deng-dev/marathon-hub-race-scraper](https://github.com/jason-deng-dev/marathon-hub-race-scraper)

**Author:** Jason Deng

**Date:** March 2026

**Status:** In Development

---

## 1. Problem Statement

### 1.1 Context

running.moximoxi.net is a marathon platform targeting Chinese runners based in China who are planning to run Japanese marathons. One of the platform's four core destinations is a **race listings hub** — a browsable, up-to-date database of upcoming Japanese marathons sourced from RunJapan.

Currently, race data is scraped manually, stored in a local `races.json` file with stale data, and not surfaced on the live WordPress site in any useful form.

### 1.2 The Problem

Race data needs to be:

- **Fresh** — upcoming marathons update regularly (new registrations open, deadlines pass, events are added)
- **Complete** — name, date, location, entry fee, description, registration URL, images
- **Accessible** — surfaced on the live WordPress site for platform users with search, filter, and detail views
- **Automated** — no human intervention required to keep the listings current

### 1.3 Goals

- Scrape RunJapan weekly and normalize race data into a clean `races.json`
- Serve race data via an Express API running on the AWS Lightsail VPS
- Build a React SPA embedded in WordPress as a plugin/shortcode — browsable race listings with search, filter, race detail view, and signup link
- CTAs on each race linking to the platform ecosystem (store, community, marathon prep tools)
- Wire the full pipeline on a weekly cron

### 1.4 Non-Goals

- WordPress custom post types or WP REST API sync — data is served directly from Express API
- Standalone portfolio frontend — the React SPA lives inside WordPress
- Real-time scraping (weekly cadence is sufficient)
- User-submitted race data
- Paid race registration or ticketing

---

## 2. Architecture

```
┌──────────────── AWS Lightsail VPS (docker-compose) ────────────────┐
│                                                                     │
│  [Scraper container]              [Race Hub container]              │
│   cron: scraper.js weekly          Express :3001 (always up)        │
│   no HTTP server                   GET /api/races                   │
│   pure cron process                GET /api/races/:id               │
│          │                         GET /api/races/upcoming          │
│          │ writes                         │                         │
│          ▼                               │ reads                    │
│    scraper/races.json  ◄─────────────────┘                          │
│    (shared volume)                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                  HTTPS
                              GET /api/races
                                    │
                                    ▼
                       [WordPress plugin on running.moximoxi.net]
                        React SPA bundled as WP plugin
                        shortcode: [race_hub]
                        fetches from Race Hub API on page load
                        renders race listing + detail views
                                    │
                                    ▼
                       running.moximoxi.net/racehub/
```

**Scraper and Race Hub are separate containers.** The Scraper is a pure cron process — no HTTP server, no persistent process. The Race Hub is a persistent Express server that reads `races.json` from the shared volume. They are decoupled: the scraper just writes a file, the Race Hub just serves it. Either can be restarted independently.

**How WordPress receives the data:** The React SPA is bundled (Vite) and registered as a WordPress plugin. When a visitor loads the race hub page, the browser makes a `GET /api/races` request directly to the Race Hub container on Lightsail. WordPress just hosts the shortcode and the bundled JS/CSS — all data and logic live on the VPS. CORS is configured on the Race Hub to allow requests from `running.moximoxi.net`.

---

## 3. Component Breakdown

#### Scraper container

- Pure cron process — no HTTP server
- Scrapes RunJapan weekly for upcoming Japanese marathon listings
- Pulls: race name, date, location, entry period, description, registration URL, images, entry status
- Writes `scraper/races.json` and `scraper/run_log.json` to the shared volume
- **Core scraping logic fully implemented in `rednote-content-automation/src/scraper.js` — port directly**

#### Race Hub container

- Persistent Express server (always up, separate from the scraper)
- Reads `scraper/races.json` from the shared volume
- Exposes `GET /api/races` with filtering query params
- Exposes `GET /api/races/:id` for race detail
- Exposes `GET /api/races/upcoming` for date-filtered listing
- CORS configured to allow requests from `running.moximoxi.net`
- Receives manual sync trigger from the dashboard (`POST /api/sync`, auth-protected)

#### races.json (shared volume)

- Lives at `scraper/races.json` on the shared Docker volume
- Written by the Scraper container, read by the Race Hub container and the XHS automation container
- Includes `last_updated` timestamp — used by dashboard to show data freshness

#### React SPA (WordPress plugin)

- Bundled React app registered as a WordPress plugin
- Embedded on running.moximoxi.net/racehub/ via shortcode
- Fetches from Express API (`GET /api/races`)
- Features: race card grid, search, filter panel, race detail view, CTAs
- No WordPress DB involvement — pure frontend reading from the API

#### Weekly cron

- Scrape → normalize → write races.json
- Runs weekly (Sunday 2am JST)
- Manual trigger: `POST /api/sync`

---

## 4. Data Design

### 4.1 Express API Endpoints

|Method|Endpoint|Description|
|---|---|---|
|`GET`|`/api/races`|All races, supports query params|
|`GET`|`/api/races/:id`|Single race detail|
|`GET`|`/api/races/upcoming`|Races with `date >= today`, sorted by date|
|`POST`|`/api/sync`|Manual pipeline trigger (requires `X-Sync-Key` header)|

**Query params for `/api/races`:**

|Param|Values|Example|
|---|---|---|
|`status`|`open`, `closed`|`?status=open`|
|`after`|ISO date string|`?after=2026-06-01`|
|`before`|ISO date string|`?before=2026-12-31`|
|`search`|Race name substring|`?search=osaka`|
|`sort`|`date_asc`, `date_desc`|`?sort=date_asc`|

---

## 5. React SPA — Feature Spec

### 5.1 Race Listing View

- Card grid of upcoming races, sorted by date ascending by default
- Each card shows: race name, date, location, entry status badge
- Search bar: race name substring match
- Filter by entry status: All / Open / Closed
- Filter by date range
- Click card → race detail view

### 5.2 Race Detail View

- Full race info: name, date, location, entry period, description, images
- Entry status badge
- **"Register Now" button** — links to `registrationUrl` (RunJapan signup page), opens in new tab
- CTA section linking to platform ecosystem:
  - Race Hub (full details + more races)
  - Store (Japanese running products)
  - Community (runners planning the same race)
  - Marathon Prep Tools

### 5.3 UI States

- Loading skeleton on initial fetch
- Empty state when filters return no results
- Error state if Express API is unreachable

### 5.4 WordPress Integration

- React app bundled with Vite, output to `wp-plugin/dist/`
- WordPress plugin registers shortcode `[race_hub]`
- Shortcode enqueues the bundled JS/CSS and renders a mount point div
- Operator adds shortcode to any WordPress page — no code changes needed

---

## 6. Technical Decisions

|Decision|Choice|Alternatives Considered|Rationale|
|---|---|---|---|
|Data delivery|Express API serving races.json|WP custom post types + WP REST API|No WP DB schema needed; scraper just updates a file; any developer can understand the flow; Express already running on Lightsail for other pipelines|
|Frontend embedding|React SPA as WordPress plugin|Standalone deployment, WP theme templates|Operator never leaves WordPress; maintainable by any WP developer; no separate hosting needed|
|Scrape target|RunJapan only|JogNote, marathon-link.com|RunJapan has the most complete race data; single source reduces complexity|
|Data store|races.json flat file|PostgreSQL, SQLite|Sufficient for ~100-200 races; zero infra overhead; easy to inspect|
|Scrape cadence|Weekly|Daily, real-time|Race data changes slowly; weekly is fresh enough; reduces RunJapan request load|
|Language|Node.js / JavaScript|Python|Consistent with rest of stack|

---

## 7. Implementation Phases

### 7.1 Current Status

|Component|Status|Notes|
|---|---|---|
|scraper.js|✅ Done (port)|Fully working in `rednote-content-automation/src/scraper.js`|
|races.json|🔧 Partial|Exists, data stale|
|Express API server|❌ Not started|New — simple file-read API|
|React SPA WordPress plugin|❌ Not started|—|
|Weekly cron|❌ Not started|—|
|Deploy|❌ Not started|—|

### 7.2 Phase 1 — Data Pipeline

1. Port `scraper.js` from `rednote-content-automation/src/scraper.js`
2. Validate `races.json` output — abort if < 30 races returned
3. Wire weekly cron

**Exit criteria:** `races.json` contains 30+ races with complete data. Cron runs cleanly weekly.

### 7.3 Phase 2 — Express API

1. Build Express API: `GET /api/races`, `GET /api/races/:id`, `GET /api/races/upcoming`
2. Add query param filtering (status, date range, search, sort)
3. Add `POST /api/sync` manual trigger with secret key auth
4. Deploy to AWS Lightsail alongside other pipelines

**Exit criteria:** API returns filtered race data from `races.json`. Manual sync trigger works.

### 7.4 Phase 3 — React SPA WordPress Plugin

1. Build React SPA: race listing, filter panel, search, detail view with signup link and CTAs
2. Bundle with Vite, output to `wp-plugin/dist/`
3. Build WordPress plugin: register shortcode, enqueue bundled assets
4. Upload plugin to running.moximoxi.net, add shortcode to race hub page
5. Smoke test end-to-end: WordPress page loads → React SPA fetches from Lightsail API → races display

**Exit criteria:** Race hub page on running.moximoxi.net shows live races with filtering, detail view, and signup links working.

---

## 8. Engineering Challenges & Solutions

### 8.1 RunJapan Has No Public API

**Challenge:** All race data must be scraped from HTML. RunJapan's markup may change without notice.

**Solution:** Scraper already handles this — selectors isolated in config, validation aborts without overwriting `races.json` if < 30 races returned. Proven in production via `rednote-content-automation`.

### 8.2 CORS for WordPress → Lightsail API

**Challenge:** The React SPA embedded in WordPress (running.moximoxi.net) will make cross-origin requests to the Express API on Lightsail.

**Solution:** Add CORS headers to the Express API allowing `running.moximoxi.net` as an origin.

---

## 9. Open Questions

- **Image handling:** Hotlink race images from RunJapan CDN, or download and proxy through the API? Hotlinking is simpler but fragile if RunJapan changes image URLs.
- **Alerts:** Should the dashboard send an alert if the weekly scrape returns < 30 races? Likely yes — add to dashboard spec.

---

## 10. Repository Structure

```
marathon-hub-race-scraper/
    ├── scraper/                    # Scraper container (pure cron, no HTTP)
    │   ├── scraper.js              #   RunJapan scraper (port from rednote-content-automation)
    │   ├── Dockerfile
    │   └── package.json
    ├── race-hub/                   # Race Hub container (persistent Express server)
    │   ├── server.js               #   Express API — GET /api/races, /api/races/:id, /api/races/upcoming
    │   ├── Dockerfile
    │   └── package.json
    └── wp-plugin/
        ├── race-hub.php            # WordPress plugin (registers [race_hub] shortcode, enqueues assets)
        └── dist/                   # Vite build output (bundled React SPA — fetches from Race Hub API)
```

**Note:** `scraper/races.json` and `scraper/run_log.json` live on the shared Docker volume at runtime — not committed to the repo.
