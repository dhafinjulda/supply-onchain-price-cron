# Price Cron Service

TypeScript Express.js service that scrapes Robusta (RM) and Arabica (KC) coffee futures prices from Barchart via Puppeteer, stores them in PostgreSQL via Prisma, and runs on a scheduled cron job with manual HTTP trigger endpoints.

## Features

- TypeScript for type safety
- Express.js REST API
- Prisma ORM for database management
- Puppeteer-based scraping of Barchart futures data (RM & KC)
- Scheduled cron jobs using node-cron (default: daily 03:00 GMT+7)
- Bearer token authentication on all trigger endpoints
- Manual HTTP endpoints to trigger individual scrapers
- IDR conversion via exchangerate-api
- 30-day moving average calculation and MA discount value generation
- Environment-based configuration
- Health check endpoint
- Graceful shutdown handling

## Tech Stack

- **TypeScript** 5.3+
- **Express.js** 4.18+
- **Prisma** 6.4.1
- **node-cron** 3.0+
- **Puppeteer** 24.0+
- **PostgreSQL** (configurable)

## Installation

```bash
npm install
```

## Configuration

Create a `.env` file based on `.env.example`:

```env
PORT=3000
CRON_TOKEN=your-secret-token-here
CRON_SCHEDULE="0 3 * * *"
CRON_TIMEZONE=Asia/Bangkok
DATABASE_URL="postgresql://user:password@localhost:5432/yourdb?schema=public"
PUPPETEER_EXECUTABLE_PATH=        # optional: path to Chromium (e.g. /usr/bin/chromium-browser in production)
```

| Variable | Default | Description |
|---|---|---|
| `PORT` | `3000` | HTTP server port |
| `CRON_TOKEN` | `default-secret-token` | Bearer token for all trigger endpoints |
| `CRON_SCHEDULE` | `0 3 * * *` | Cron expression (runs once daily at 03:00) |
| `CRON_TIMEZONE` | `Asia/Bangkok` | Timezone for cron schedule (GMT+7) |
| `DATABASE_URL` | — | PostgreSQL connection string |
| `PUPPETEER_EXECUTABLE_PATH` | — | Custom Chromium path (production) |

### Cron Schedule Format

The `CRON_SCHEDULE` uses standard cron syntax and is evaluated in the `CRON_TIMEZONE` timezone:
```
* * * * *
│ │ │ │ │
│ │ │ │ └─── Day of week (0-7, Sunday = 0 or 7)
│ │ │ └───── Month (1-12)
│ │ └─────── Day of month (1-31)
│ └───────── Hour (0-23)
└─────────── Minute (0-59)
```

Examples:
- `0 3 * * *` - Daily at 03:00 (default, GMT+7 via `Asia/Bangkok`)
- `0 0 * * *` - Daily at midnight
- `0 9 * * 1` - Every Monday at 9 AM

The cron fires **once per day** by default. The scraper itself also deduplicates by `tradeDate`, so re-running on the same day is safe.

## Usage

### Build the project

```bash
npm run build
```

### Start the server (production)

```bash
npm start
```

### Development mode (with auto-reload)

```bash
npm run dev
```

## API Endpoints

All `POST` endpoints require a Bearer token:
```
Authorization: Bearer your-secret-token-here
```

**Unauthorized response (all endpoints):**
```json
{ "success": false, "message": "Unauthorized: Invalid or missing token" }
```

---

### POST /cron

Runs both `scrapeRm` and `scrapeKC` sequentially (same as the scheduled job).

```json
{ "success": true, "message": "Cron job executed successfully", "timestamp": "..." }
```

### POST /scrape/rm

Manually trigger Robusta (RM) scrape only.

```json
{ "success": true, "message": "scrapeRm executed successfully", "timestamp": "..." }
```

### POST /scrape/kc

Manually trigger Arabica (KC) scrape only.

```json
{ "success": true, "message": "scrapeKC executed successfully", "timestamp": "..." }
```

### GET /health

Health check — no auth required.

```json
{
  "status": "ok",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "cronSchedule": "0 3 * * *",
  "cronTimezone": "Asia/Bangkok"
}
```

## Testing

```bash
# Trigger full cron task (RM + KC)
curl -X POST http://localhost:3000/cron \
  -H "Authorization: Bearer your-secret-token-here"

# Trigger Robusta scrape only
curl -X POST http://localhost:3000/scrape/rm \
  -H "Authorization: Bearer your-secret-token-here"

# Trigger Arabica scrape only
curl -X POST http://localhost:3000/scrape/kc \
  -H "Authorization: Bearer your-secret-token-here"

# Health check
curl http://localhost:3000/health
```

## Project Structure

```
price-cron/
├── src/
│   ├── index.ts              # Express server, cron schedule, endpoints
│   ├── lib/
│   │   └── price-index.ts    # PriceIndexService (scrapeRm, scrapeKC)
│   └── server/
│       └── db.ts             # Prisma client singleton
├── prisma/
│   └── schema.prisma         # Prisma database schema
├── dist/                     # Compiled TypeScript output
├── package.json
├── tsconfig.json
├── .env
└── .env.example
```

## How It Works

1. On schedule (or manual trigger), `performCronTask()` calls `scrapeRm()` then `scrapeKC()` — each failure is caught independently so one won't block the other.
2. Each scraper launches a headless Chromium via Puppeteer, navigates to Barchart, and intercepts the `/proxies/core-api/v1/quotes/get` API response to extract the active contract's OHLCV data.
3. IDR conversion is fetched from `exchangerate-api.com` (falls back to 16,000 if unavailable).
4. A 30-day moving average is computed from the last 30 `MarketData` records and stored alongside the price.
5. `MaDiscountValue` records are generated automatically for any configured `MaDiscountSetting` entries.
