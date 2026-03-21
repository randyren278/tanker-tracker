# Tanker Tracker — Production Deployment Guide

## Architecture

The app has **two runtime components** that deploy separately:

```
┌─────────────────────────────────────────────────────────────────┐
│                         VERCEL                                  │
│                                                                 │
│  Next.js 16 App (Frontend + API Routes)                        │
│  ├── Pages: /dashboard, /fleet, /analytics, /about             │
│  ├── API Routes: /api/vessels, /api/anomalies, /api/status...  │
│  └── Auth: JWT session cookies                                  │
│                                                                 │
│  Reads from DB — only writes watchlist/alerts                   │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           │  DATABASE_URL (pooled connection)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    TIMESCALE CLOUD                              │
│                                                                 │
│  PostgreSQL + TimescaleDB                                      │
│  ├── vessel_positions (hypertable, 1-day chunks, compression)  │
│  ├── vessels, vessel_sanctions, vessel_anomalies               │
│  ├── oil_prices, news_items, alerts, watchlist                 │
│  └── vessel_risk_scores, vessel_proximity_events               │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           │  DATABASE_URL (direct connection)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                       RAILWAY                                   │
│                                                                 │
│  AIS Ingester Service (standalone Node.js process)             │
│  ├── Persistent WebSocket to AISStream.io                      │
│  ├── Cron: anomaly detection (every 15/30 min)                 │
│  ├── Cron: oil prices (6h), news (30m), sanctions (daily)      │
│  └── Writes positions, upserts vessels, runs all detections    │
└─────────────────────────────────────────────────────────────────┘
```

### Why This Split

| Component | Why it can't run on Vercel |
|---|---|
| **AIS Ingester** | Needs a persistent WebSocket connection to AISStream.io. Vercel functions have a 60s max execution time. |
| **Cron Jobs** | Detection runs every 15–30 min with heavy SQL queries. Vercel Cron has a 60s limit per invocation. |
| **Data Refresh** | Sanctions CSV download + parse can take 30+ seconds. Prices/news need periodic polling. |

The Next.js frontend + API routes are purely read-heavy — perfect for Vercel serverless.

---

## Step 1: Database — Timescale Cloud

The schema uses TimescaleDB hypertables, compression policies, and `create_hypertable()`. **You need Timescale Cloud, not plain PostgreSQL.** Neon/Supabase won't work without removing TimescaleDB features.

### Setup

1. Go to [cloud.timescale.com](https://cloud.timescale.com) → Create Service
2. Pick **Free Tier** (30-day trial, 25 GB) or **Essential** (~$22/mo for production)
3. Region: **us-east-1** (closest to Vercel and Railway default regions)
4. Note both connection strings:
   - **Pooled** (port 5433): for Vercel — handles many short-lived serverless connections
   - **Direct** (port 5432): for Railway — single long-lived process

### Apply Schema

```bash
# Replace with your Timescale Cloud direct connection string
export DATABASE_URL="postgresql://tsdbadmin:PASSWORD@HOST:5432/tsdb?sslmode=require"

psql "$DATABASE_URL" -f src/lib/db/schema.sql
```

You should see output confirming table creation and hypertable setup. If you get errors about the TimescaleDB extension, run this first:

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

---

## Step 2: Vercel — Next.js Frontend

### 2a. Pre-Deploy Fix: bcrypt

Vercel serverless has issues with `bcrypt`'s native C++ bindings. Switch to the pure-JS version:

```bash
npm uninstall bcrypt @types/bcrypt
npm install bcryptjs
npm install -D @types/bcryptjs
```

Then update two files:

**`src/lib/auth.ts`** — change line 1:
```ts
// Before
import bcrypt from 'bcrypt';
// After
import bcrypt from 'bcryptjs';
```

**`src/lib/auth.test.ts`** — change line 2:
```ts
// Before
import bcrypt from 'bcrypt';
// After
import bcrypt from 'bcryptjs';
```

### 2b. Add vercel.json

Create `vercel.json` in the project root:

```json
{
  "framework": "nextjs",
  "regions": ["iad1"],
  "functions": {
    "src/app/api/**/*.ts": {
      "maxDuration": 30
    }
  }
}
```

- `regions: ["iad1"]` — US East, close to your DB
- `maxDuration: 30` — some DB queries need more than the default 10s on cold start

### 2c. Lower Connection Pool Size

Vercel serverless creates a new pool per function invocation. Edit `src/lib/db/index.ts`:

```ts
export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 3,               // was 20 — lower for serverless
  idleTimeoutMillis: 10000,  // was 30000 — release faster
  connectionTimeoutMillis: 5000,
});
```

### 2d. Deploy

1. Push all changes to GitHub
2. Go to [vercel.com](https://vercel.com) → **Add New Project** → Import `tanker-tracker`
3. Vercel auto-detects Next.js — no build settings needed
4. Add environment variables:

| Variable | Value |
|---|---|
| `DATABASE_URL` | Your Timescale Cloud **pooled** connection string (port 5433) |
| `NEXT_PUBLIC_MAPBOX_TOKEN` | Your Mapbox public token (`pk.eyJ...`) |
| `JWT_SECRET` | A random 32+ character string (e.g., `openssl rand -hex 32`) |
| `PASSWORD_HASH` | A bcrypt hash of your login password |

Generate the password hash:
```bash
node -e "import('bcryptjs').then(b => b.default.hash('YOUR_PASSWORD', 10).then(h => console.log(h)))"
```

5. Click **Deploy**

### 2e. Verify Vercel

- [ ] Site loads at `your-project.vercel.app`
- [ ] Login page works (enter password → redirects to dashboard)
- [ ] Map renders (Mapbox token working)
- [ ] No console errors about missing env vars
- [ ] `/api/status` returns JSON (even if sources show "offline" — DB is empty until ingester runs)

---

## Step 3: Railway — AIS Ingester

### 3a. Deploy

1. Go to [railway.app](https://railway.app) → **New Project** → **Deploy from GitHub Repo** → Select `tanker-tracker`
2. Railway will detect Node.js and install dependencies from root `package.json`
3. Go to **Settings** → **Deploy** section:
   - **Custom Start Command**: `npx tsx src/services/ais-ingester/index.ts`
   - **Restart Policy**: On Failure (max 10 retries)
4. Add environment variables:

| Variable | Value |
|---|---|
| `DATABASE_URL` | Your Timescale Cloud **direct** connection string (port 5432) |
| `AISSTREAM_API_KEY` | Your key from [aisstream.io](https://aisstream.io) |
| `ALPHA_VANTAGE_API_KEY` | Your key from [alphavantage.co](https://www.alphavantage.co/support/#api-key) |
| `FRED_API_KEY` | Your key from [fred.stlouisfed.org](https://fred.stlouisfed.org/docs/api/api_key.html) |
| `NEWSAPI_KEY` | Your key from [newsapi.org](https://newsapi.org/register) |

5. Deploy

### 3b. Verify Railway

Check the Railway logs. You should see:

```
============================================================
AIS Ingester Service
============================================================
Environment: production
Database URL: (configured)
AISStream API Key: (configured)
============================================================
Connecting to AISStream.io...
Connected. Sending subscription...
Subscription sent. Waiting for messages...
Starting anomaly detection cron jobs...
Detection cron jobs scheduled:
  - going_dark: every 15 minutes
  - loitering/speed/deviation/repeat_dark/sts: every 30 minutes
Starting background refresh jobs...
Refresh cron jobs scheduled: prices every 6h, news every 30m, sanctions daily
[STARTUP] Prices fetched: 2 symbols
[STARTUP] News fetched: 10 headlines
[STARTUP] Sanctions fetched: 847 entries upserted, 0 stale removed
Processed 142 messages in the last minute
```

If you see `WebSocket error` or `Connection closed`, check:
- Is `AISSTREAM_API_KEY` correct?
- Is the Railway service region not blocked by AISStream?

### 3c. Verify End-to-End

Once Railway is running for a few minutes:

```bash
# Check positions are flowing
curl https://your-site.vercel.app/api/status
# Expected: {"ais":"live","prices":"live","news":"live"}

# Check vessel count
curl https://your-site.vercel.app/api/vessels | jq '.vessels | length'
# Expected: > 0
```

---

## Step 4: Custom Domain (Optional)

1. In Vercel dashboard → **Settings** → **Domains** → Add your domain
2. Add DNS records as Vercel instructs (usually a CNAME to `cname.vercel-dns.com`)
3. Vercel auto-provisions SSL
4. Railway doesn't need a public domain — it's a background worker

---

## Environment Variables — Complete Reference

| Variable | Vercel | Railway | How to Get |
|---|---|---|---|
| `DATABASE_URL` | ✅ pooled | ✅ direct | Timescale Cloud dashboard |
| `NEXT_PUBLIC_MAPBOX_TOKEN` | ✅ | — | [mapbox.com](https://account.mapbox.com/access-tokens/) |
| `JWT_SECRET` | ✅ | — | `openssl rand -hex 32` |
| `PASSWORD_HASH` | ✅ | — | `bcryptjs.hash('password', 10)` |
| `AISSTREAM_API_KEY` | — | ✅ | [aisstream.io](https://aisstream.io) (free account) |
| `ALPHA_VANTAGE_API_KEY` | — | ✅ | [alphavantage.co](https://www.alphavantage.co/support/#api-key) |
| `FRED_API_KEY` | — | ✅ | [fred.stlouisfed.org](https://fred.stlouisfed.org/docs/api/api_key.html) |
| `NEWSAPI_KEY` | — | ✅ | [newsapi.org](https://newsapi.org/register) |

---

## Cost Estimate

| Service | Plan | Monthly Cost |
|---|---|---|
| Vercel | Hobby (free) or Pro ($20) | $0 – $20 |
| Railway | Hobby | $5 |
| Timescale Cloud | Free tier or Essential | $0 – $22 |
| AISStream.io | Free (1 connection) | $0 |
| Mapbox | Free (50k map loads/mo) | $0 |
| Alpha Vantage | Free (25 req/day) | $0 |
| FRED | Free | $0 |
| NewsAPI | Free (100 req/day) | $0 |
| **Total** | | **$5 – $47/mo** |

**Minimum viable: $5/mo** (Vercel free + Railway hobby + Timescale free trial)

---

## Monitoring

### Railway (Ingester)
- Check logs for WebSocket disconnects and cron errors
- The ingester auto-reconnects on disconnect (5s delay)
- Duplicate cron registration is guarded (idempotent `started` flag)

### Vercel (Frontend)
- Vercel dashboard shows function invocations, errors, and latency
- Monitor cold start times on `/api/vessels` (heaviest query)

### Health Check
```bash
# Quick status check — should return "live" for all three sources
curl https://your-site.vercel.app/api/status
```

If any source shows "offline":
- `ais: offline` → Check Railway logs, AISStream connection
- `prices: offline` → Alpha Vantage rate limit hit, FRED fallback also failed
- `news: offline` → NewsAPI key expired or rate limited

### Database Storage
- Vessel positions grow ~50MB/day (compressed by TimescaleDB after 7 days)
- Free tier: 25GB → ~500 days before you need to purge or upgrade
- Monitor in Timescale Cloud dashboard

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| Map blank, no vessels | DB empty, ingester not running | Check Railway is deployed and running |
| `500` on API routes | DB connection failed | Check `DATABASE_URL` on Vercel, ensure pooled endpoint |
| `bcrypt` build error on Vercel | Native module incompatible | Switch to `bcryptjs` (see Step 2a) |
| Vessels disappear after 48h | Position window is 48h | Expected behavior — vessels reappear when they transmit again |
| "missed execution" cron warning | Old bug (fixed) | Ensure you deployed latest code with idempotent cron guards |
| Login doesn't work | Wrong `PASSWORD_HASH` or `JWT_SECRET` | Regenerate hash, ensure env vars match |
| Railway keeps restarting | Missing env var or DB unreachable | Check Railway logs for the specific error |
