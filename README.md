# Marathon Hub — Race Scraper Pipeline

A fully automated pipeline that scrapes upcoming Japanese marathon data from RunJapan daily, normalizes it into a clean dataset, syncs it to a live WordPress site via the REST API, and serves it through a standalone Express API powering a React frontend.

Built as part of [running.moximoxi.net](https://running.moximoxi.net) — a marathon platform for Chinese runners planning Japanese races.

## What It Does

- Scrapes 50+ upcoming Japanese marathons from RunJapan daily (name, date, location, distance, entry fee, description, registration URL, images)
- Normalizes race data: date formatting, distance categorization, entry status derivation, deduplication
- Syncs to WordPress as custom post types via the WP REST API (idempotent — safe to re-run daily)
- Serves race data via a standalone Express API with filtering, search, and sort
- React SPA frontend for browsing and filtering races — deployable independently of WordPress

## Stack

- Node.js / Express
- Cheerio (HTML scraping)
- PostgreSQL
- WordPress REST API + Application Passwords
- React (Vite) + Tailwind CSS
- node-cron

## Architecture

See [docs/design-doc.md](docs/design-doc.md) for full system architecture, data schema, technical decisions, and engineering challenges.

## Running Locally
```bash
git clone https://github.com/jason-deng-dev/marathon-hub-race-scraper
cd marathon-hub-race-scraper
npm install
cp .env.example .env
# Add your WP credentials to .env
node scraper.js
```

## API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| GET | /api/races | All races, supports query params |
| GET | /api/races/:id | Single race by ID |
| GET | /api/races/upcoming | Upcoming races sorted by date |
| POST | /api/sync | Manual pipeline trigger |

## Status

In development — see [docs/checklist.md](docs/checklist.md) for current progress.