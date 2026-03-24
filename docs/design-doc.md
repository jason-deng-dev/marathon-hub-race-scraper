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

Currently, race data is scraped manually or via a broken cron, stored in a local `races.json` file with only 5 races and stale data. There is no live connection between the scraper and the WordPress site. The race listings page on the platform is effectively non-functional.

### 1.2 The Problem

Race data needs to be:

- **Fresh** — upcoming marathons update regularly (new registrations open, deadlines pass, events are added)
- **Complete** — name, date, location, distance, entry fee, description, registration URL, images
- **Accessible** — surfaced on the live WordPress site for platform users, and in a standalone portfolio frontend for job search purposes
- **Automated** — no human intervention required to keep the listings current

The scraper exists but is broken. The WordPress plugin exists but is not deployed. There is no end-to-end pipeline connecting any of these pieces.

### 1.3 Goals

- Scrape RunJapan daily and normalize race data into a clean `races.json`
- Sync race data to WordPress as custom post types via the WP REST API
- Serve race data via a standalone Express API backend
- Build a React SPA frontend over the Express API as a polished portfolio artifact
- Wire the full pipeline on a daily cron with an on-demand manual trigger

### 1.4 Non-Goals

- Real-time scraping (daily cadence is sufficient for race data)
- User-submitted race data (RunJapan is the sole source in v1)
- Paid race registration or ticketing
- Scraping sources other than RunJapan in v1

---

## 2. Dual-Output Architecture

This pipeline has two distinct consumers of the same underlying data:

|Output|Purpose|Audience|
|---|---|---|
|**WordPress integration**|Live production race listings on running.moximoxi.net|Chinese runners using the platform|
|**Portfolio Express/React frontend**|Standalone deployable app, independent of WP|Hiring managers, portfolio reviewers|

Both outputs consume from the same `races.json` source. The scraper and normalizer are shared infrastructure. The WordPress sync and the Express API are parallel consumers — neither depends on the other.

This separation is intentional: if the WordPress site goes down or the job ends, the portfolio frontend continues to work independently.

---

## 3. System Architecture

### 3.1 High-Level Overview

```
SCRAPE → NORMALIZE → STORE → DISTRIBUTE
                                ├── WordPress (production)
                                └── Express API → React SPA (portfolio)
```

### 3.2 Component Breakdown

#### scraper.js (complete — port from rednote-content-automation)

- Scrapes RunJapan for upcoming Japanese marathon listings
- Pulls: race name, date, location, prefecture, distance, entry fee, description, registration URL, images
- Normalizes raw HTML into structured race objects
- Writes to `races.json`
- Triggered by daily cron and manual `/api/sync` endpoint
- **Implementation complete and proven in `rednote-content-automation/src/scraper.js` — port directly rather than rebuilding**

#### normalizer.js (new)

- Cleans and standardizes scraped data (date formats, distance normalization, null handling)
- Detects entry status: open / closed / lottery based on deadline fields
- Deduplicates races by name + date to prevent duplicate WP posts on re-runs

#### races.json (exists, central data store)

- Single source of truth for all downstream consumers
- Written by scraper, read by WP sync and Express API
- Includes `last_updated` timestamp

#### wp-sync.js (exists as manual script, needs wiring)

- Reads `races.json`
- Pushes race data to WordPress via WP REST API
- Creates new custom post types for new races
- Updates existing posts if data has changed (compare by race name + date)
- Skips races that are identical to existing WP posts (idempotent)

#### WordPress plugin (exists, not deployed)

- Registers `race` custom post type in WordPress
- Exposes authenticated REST API endpoint for external writes
- Renders race listings on the front-end hub page

#### Express API server (exists at localhost:3000, needs productionizing)

- Reads from `races.json`
- Exposes REST endpoints consumed by the React SPA
- Also exposes `/api/sync` manual trigger endpoint (auth-protected)

#### React SPA frontend (not started)

- Consumes Express API
- Full race listings UI with filtering, search, and sort
- Deployable independently of WordPress

#### Cron orchestrator

- Daily schedule: `scrape → normalize → write races.json → wp-sync → log`
- Manual trigger: `POST /api/sync` with secret key header
- Logs each stage to `pipeline.log`

### 3.3 Data Flow

```
RunJapan website
    ↓  (HTTP scrape, daily cron + manual trigger)
scraper.js
    ↓  (normalizes + deduplicates)
normalizer.js
    ↓  (writes)
races.json
    ├──→  wp-sync.js
    │         ↓  (WP REST API authenticated POST)
    │     WordPress custom post types
    │         ↓
    │     running.moximoxi.net/races/  (production hub page)
    │
    └──→  Express API server
              ↓  (JSON endpoints)
          React SPA
              ↓
          Portfolio frontend (standalone deployment)
```

---

## 4. Data Design

### 4.1 races.json Schema

```json
{
  "races": [
    {
      "id": "tokyo-marathon-2026",
      "name": "東京マラソン2026",
      "name_en": "Tokyo Marathon 2026",
      "date": "2026-03-01",
      "location": "Tokyo",
      "prefecture": "東京都",
      "distance": "42.195km",
      "distance_category": "full",
      "entry_fee": "¥15,700",
      "entry_status": "closed",
      "registration_deadline": "2025-10-31",
      "registration_url": "https://www.marathon.tokyo/",
      "description": "...",
      "images": ["https://runjapan.com/images/tokyo-2026.jpg"],
      "source_url": "https://runjapan.com/races/tokyo-marathon-2026",
      "scraped_at": "2026-03-17T02:00:00Z"
    }
  ],
  "last_updated": "2026-03-17T02:00:00Z",
  "total_races": 47
}
```

### 4.2 Distance Category Normalization

|Raw value|Normalized category|
|---|---|
|42.195km, フルマラソン|`full`|
|21.0975km, ハーフ|`half`|
|10km, 10K|`10k`|
|5km, 5K|`5k`|
|Other / unknown|`other`|

### 4.3 Entry Status Logic

Derived at normalization time, not scraped directly:

```
if registration_deadline < today → "closed"
if registration_deadline > today AND entry_method === "lottery" → "lottery"
if registration_deadline > today AND entry_method === "first-come" → "open"
if deadline unknown → "unknown"
```

### 4.4 Express API Endpoints

|Method|Endpoint|Description|
|---|---|---|
|`GET`|`/api/races`|All races, supports query params|
|`GET`|`/api/races/:id`|Single race by ID|
|`GET`|`/api/races/upcoming`|Races with `date >= today`, sorted by date|
|`POST`|`/api/sync`|Manual pipeline trigger (requires `X-Sync-Key` header)|

**Query params for `/api/races`:**

|Param|Values|Example|
|---|---|---|
|`distance`|`full`, `half`, `10k`, `5k`|`?distance=full`|
|`prefecture`|Prefecture name or code|`?prefecture=Tokyo`|
|`status`|`open`, `closed`, `lottery`|`?status=open`|
|`after`|ISO date string|`?after=2026-06-01`|
|`before`|ISO date string|`?before=2026-12-31`|
|`search`|Race name substring|`?search=osaka`|
|`sort`|`date_asc`, `date_desc`|`?sort=date_asc`|

---

## 5. WordPress Sync — Technical Decision

### 5.1 Options Considered

|Approach|Pros|Cons|
|---|---|---|
|**WP REST API (authenticated POST)**|Standard, well-documented, works over HTTP, no server access needed|Requires Application Password setup, slightly more round-trips|
|Direct DB write|Fast, no HTTP overhead|Tightly coupled to DB schema, dangerous, bypasses WP hooks|
|WP CLI / shell script|Simple for one-off runs|Requires SSH access, not portable, hard to call from Node|

### 5.2 Decision: WP REST API with Application Passwords

Use the WordPress REST API (`/wp-json/wp/v2/race`) with Application Passwords for authentication. The WordPress plugin registers the `race` custom post type and its meta fields, making them available via the REST API. `wp-sync.js` authenticates as an admin user and POSTs race data.

**Rationale:**

- Standard WordPress pattern — well documented, maintainable
- Works over HTTPS from any Node process without server access
- Idempotent: check for existing post by race slug before creating vs updating
- Authentication via Application Passwords (WP 5.6+) — no plugin required, revocable

### 5.3 Idempotency Strategy

On each sync run:

1. Fetch all existing `race` post slugs from WP REST API
2. For each race in `races.json`: if slug exists → `PUT` update; if not → `POST` create
3. Do not delete WP posts for races no longer in `races.json` (races may be removed from RunJapan but still valid for platform users)

---

## 6. React SPA Frontend

### 6.1 Overview

A standalone React application consuming the Express API. Deployed independently of WordPress. Designed to demonstrate full-stack capability for hiring purposes — the same data powering the production platform, surfaced through a clean, well-engineered UI.

### 6.2 Feature Spec

**Race listing page (main view):**

- Card grid of upcoming races, sorted by date ascending by default
- Each card shows: race name, date, location, distance badge, entry status badge, entry fee
- Click card → race detail view

**Filtering panel:**

- Filter by distance: All / Full / Half / 10K / 5K
- Filter by prefecture/region: dropdown of all prefectures in dataset
- Filter by entry status: All / Open / Lottery / Closed
- Date range picker: from / to
- Search bar: race name substring match (client-side)
- Sort: Date (asc/desc)
- All filters combinable, applied client-side after initial data fetch
- "Clear all filters" reset button

**Race detail view:**

- Full race info: name, date, location, description, entry fee, distance
- Entry status badge with deadline if available
- Registration link (opens RunJapan source in new tab)
- Race image if available
- Back to listings navigation

**UI state handling:**

- Loading skeleton on initial fetch
- Empty state when filters return no results
- Error state if API is unreachable

### 6.3 Stack

|Layer|Choice|Rationale|
|---|---|---|
|Frontend|React (Vite)|Fast dev server, clean build output, consistent with TOP curriculum|
|Styling|Tailwind CSS|Utility-first, rapid UI, portfolio-friendly output|
|API client|fetch / axios|Simple REST, no GraphQL overhead needed|
|Backend|Express + Node.js|Consistent with rest of stack|
|Data|races.json (file read)|Lowest coupling — no DB required for portfolio deployment|
|Deployment|Railway / Render|Simple Node + static hosting, free tier available|

---

## 7. Technical Decisions

|Decision|Choice|Alternatives Considered|Rationale|
|---|---|---|---|
|Scrape target|RunJapan only|JogNote, marathon-link.com|RunJapan has the most complete English+Japanese race data; single source reduces scraper complexity|
|Data store|races.json flat file|PostgreSQL, SQLite|Sufficient for ~100-200 races; zero infra overhead; easy to inspect and debug; portable between WP sync and Express|
|WP integration|REST API + Application Passwords|Direct DB, WP CLI|Standard, documented, no SSH dependency, revocable auth|
|Cron pattern|Daily 2am JST + manual `/api/sync`|On-demand only, real-time|Race data changes slowly; daily is fresh enough; manual trigger handles urgent updates and dev debugging|
|Frontend framework|React SPA|EJS server-render, Next.js|React demonstrates modern frontend skills; SPA is appropriate for filtering-heavy UI; consistent with TOP curriculum|
|Portfolio deployment|Standalone Express + React|WordPress-only|WordPress site could go down post-employment; portfolio must be self-contained and independently deployable|
|Language|Node.js / JavaScript|Python|Consistent with rest of stack; no context switching|

---

## 8. Implementation Phases

### 8.1 Current Status

|Component|Status|Notes|
|---|---|---|
|scraper.js|✅ Done (port)|Fully implemented and working in `rednote-content-automation/src/scraper.js` — port to this repo|
|races.json|🔧 Partial|Exists, 5 races, data 12+ days stale, cron was broken (now fixed)|
|normalizer.js|❌ Not started|Normalization currently inline in scraper, needs extraction|
|wp-sync.js|🔧 Partial|Exists as manual script, not wired to cron, not tested in production|
|WordPress plugin|🔧 Partial|Exists, not deployed to running.moximoxi.net|
|Express API server|🔧 Partial|Running at localhost:3000, not productionized, endpoints not fully implemented|
|React SPA frontend|❌ Not started|—|
|Cron orchestration|🔧 Partial|Cron exists for scraper, broken after folder rename, not wired end-to-end|
|Manual sync trigger|❌ Not started|`/api/sync` endpoint not implemented|

### 8.2 Phase 1 — Fix the Data Pipeline

1. Port `scraper.js` from `rednote-content-automation/src/scraper.js` — fully working, no rebuild needed
2. Extract `normalizer.js` from scraper: date normalization, distance categorization, entry status derivation, deduplication
4. Validate `races.json` output against schema
5. Fix and test daily cron end-to-end

**Exit criteria:** `races.json` contains 40+ races with complete, normalized data. Cron runs cleanly at 2am JST and updates the file daily.

### 8.3 Phase 2 — WordPress Integration

1. Set up Application Password on running.moximoxi.net WordPress admin
2. Deploy and activate WordPress plugin (registers `race` custom post type)
3. Test REST API endpoint (`/wp-json/wp/v2/race`) manually
4. Update `wp-sync.js` to use REST API with idempotency logic
5. Wire wp-sync into daily cron: `scrape → normalize → wp-sync`
6. Validate race listings page on running.moximoxi.net

**Exit criteria:** Race listings page on running.moximoxi.net shows 40+ current races. Re-running sync does not create duplicate posts.

### 8.4 Phase 3 — Express API + React Frontend

1. Productionize Express API: implement all `/api/races` endpoints with query param filtering
2. Implement `/api/sync` manual trigger with secret key auth
3. Build React SPA: listing page, filter panel, detail view, loading/error/empty states
4. Add `pipeline.log` logging across all stages
5. Deploy Express API + React frontend to Railway or Render

**Exit criteria:** Portfolio frontend is live at a public URL, independently of WordPress. All filters work. Manual sync trigger works.

---

## 9. Engineering Challenges & Solutions

### 9.1 RunJapan Has No Public API

**Challenge:** All race data must be scraped from HTML. RunJapan's markup may change without notice, breaking the scraper silently.

**Solution:** Isolate all CSS selectors in a single `SELECTORS` config object at the top of `scraper.js`. When RunJapan changes its markup, only the selectors need updating, not the scraper logic. Add validation after each scrape run: if fewer than 20 races are returned, flag as a potential selector failure and log a warning before writing to `races.json`.

### 9.2 Stale or Incomplete Data

**Challenge:** RunJapan race pages sometimes have missing descriptions, no entry fees listed, or registration deadlines not yet published.

**Solution:** Normalizer treats all fields as nullable. Missing fields are stored as `null`, not omitted. UI handles null gracefully — entry fee shows "TBD", deadline shows "Not yet announced". This prevents scraper failures from blocking the pipeline.

### 9.3 WordPress Sync Idempotency

**Challenge:** Daily re-runs must not create duplicate race posts in WordPress.

**Solution:** Each race gets a deterministic slug: `{race-name-slug}-{YYYY-MM-DD}` (e.g. `tokyo-marathon-2026-03-01`). Before creating a post, wp-sync checks if a post with that slug already exists. If yes → update. If no → create. This makes every sync run safe to re-run without manual cleanup.

### 9.4 Portfolio Frontend Independence

**Challenge:** The portfolio app must remain live and usable even if the production WordPress site goes down or the job ends.

**Solution:** The React SPA reads directly from the Express API, which reads from `races.json`. The Express API is deployed independently of WordPress on a separate host (Railway/Render). The only dependency is the Express server staying live. If the daily cron for RunJapan scraping also runs on the same server, the data stays fresh without any WordPress involvement.

### 9.5 Bot Detection on RunJapan

**Challenge:** Daily automated scraping may trigger rate limiting or bot detection.

**Solution:** Add randomized delays between page requests (1.5–3s), set a realistic User-Agent header, and scrape during off-peak hours (2am JST). If RunJapan returns a non-200 status or unexpected page structure, abort the run and log the failure without overwriting the existing `races.json` — preserving the last good dataset.

---

## 10. Cron & Ops Design

### 10.1 Daily Cron Schedule

```
0 2 * * *   node scraper.js && node normalizer.js && node wp-sync.js
```

Runs at 2am JST daily. Each stage is chained — if scraper fails, wp-sync does not run. All output logged to `pipeline.log`.

### 10.2 Manual Sync Trigger

```
POST /api/sync
Headers: { "X-Sync-Key": "<secret>" }
```

Triggers the full pipeline on demand. Returns:

```json
{
  "status": "ok",
  "races_scraped": 47,
  "wp_created": 3,
  "wp_updated": 12,
  "duration_ms": 4231
}
```

Use cases:

- Forcing a re-sync after fixing a parsing bug
- Pushing a newly announced major race (Tokyo, Osaka) live immediately
- Testing the pipeline in staging without waiting for 2am

### 10.3 Failure Handling

|Failure|Behaviour|
|---|---|
|RunJapan returns <20 races|Log warning, abort, do not overwrite races.json|
|RunJapan unreachable|Log error, abort, do not overwrite races.json|
|WP REST API auth failure|Log error, skip WP sync, races.json still updated|
|WP REST API timeout|Retry once after 5s, then log and skip|
|React frontend API unreachable|Show error state UI, do not crash|

---

## 11. Portfolio & Resume Framing

### 11.1 What This Project Demonstrates

This project is a production system with real users, not a bedroom project. The engineering decisions map directly to real-world backend and fullstack work:

- **Automated data pipeline** — scheduled scraping, normalization, multi-destination sync running in production daily
- **REST API design** — Express API with filtering, pagination, and authenticated admin endpoints
- **WordPress integration** — custom plugin development, WP REST API, custom post types — a common enterprise CMS integration pattern
- **React SPA** — filtering, search, and multi-state UI (loading, error, empty, populated)
- **Ops thinking** — idempotent syncs, failure handling, manual triggers, structured logging

### 11.2 Talking Points for Interviews

**"Tell me about a project you built end-to-end."**

> Built a fully automated race data pipeline for a live marathon platform. Scrapes RunJapan daily, normalizes and deduplicates ~50 races, syncs to WordPress via the REST API, and serves the same data through a separate Express API powering a React frontend. Designed for two independent consumers from a single data source — production CMS and a standalone portfolio app.

**"How did you handle reliability?"**

> The scraper validates output before writing — if it returns fewer than 20 races, we assume a selector failure and abort without overwriting the last good dataset. WP sync is idempotent: every race gets a deterministic slug, so daily re-runs don't create duplicates. Failures at each stage are isolated — a WP sync failure doesn't prevent the races.json from updating.

**"Why did you choose this stack?"**

> Node.js throughout for consistency. Flat file (races.json) instead of a database because race data is small, read-heavy, and benefits from being inspectable — you can open the file and immediately see the state of the system. WP REST API over direct DB access because it works over HTTP without server access and is the documented, maintainable integration path.

---

## 12. Open Questions

- **RunJapan scraper robustness:** How frequently does RunJapan's markup change? Should we add a Slack/email alert when the scraper returns suspiciously few races?
- **Image handling:** Should race images be downloaded and hosted locally, or hotlinked from RunJapan? Hotlinking is simpler but fragile.
- **WP plugin deployment:** What's the deployment process for the plugin — FTP, WP admin upload, or server SSH? Needs confirming with hosting setup.
- **Portfolio hosting:** Railway vs Render vs Fly.io — compare free tier persistence and cold start behaviour for the Express server.
- **Expanding beyond RunJapan:** In v2, is there value in adding JogNote or marathon-link.com as additional sources? Would require a source field in races.json and dedup logic across sources.
- **Race detail pages on WP:** Should each WP race post have a standalone page, or is a single hub page with all races sufficient for v1?

---

## 13. Repository Structure

```
marathon-hub-race-scraper/
    ├── scraper.js              # RunJapan scraper (port from rednote-content-automation)
    ├── normalizer.js           # Data normalization + dedup (new)
    ├── wp-sync.js              # WordPress REST API sync (exists, needs wiring)
    ├── races.json              # Central data store (exists, stale)
    ├── pipeline.log            # Stage-by-stage run log (new)
    ├── wp-plugin/              # WordPress custom post type plugin (exists, not deployed)
    │   └── race-listings.php
    └── frontend/               # Portfolio React SPA (not started)
        ├── server/
        │   └── index.js        # Express API server
        └── client/
            └── src/
                ├── App.jsx
                ├── components/
                │   ├── RaceCard.jsx
                │   ├── RaceDetail.jsx
                │   ├── FilterPanel.jsx
                │   └── SearchBar.jsx
                └── hooks/
                    └── useRaces.js
```

---

