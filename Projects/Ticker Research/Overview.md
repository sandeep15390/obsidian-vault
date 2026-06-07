# Ticker Research

## Overview

A web application for researching stocks before placing options trades. Given a ticker symbol it shows: current price and company info, analyst consensus (donut chart), price targets (low/mean/high range bar with upside %), 90-day price history, and recent analyst rating changes.

This is the first component of the broader options trading application. The goal is to answer "what do analysts think about this stock right now?" before entering a position.

## Status

**MVP complete — running on melody-beast.**

- Backend: `http://melody-beast.tailc98a25.ts.net:8000`
- Frontend: `http://melody-beast.tailc98a25.ts.net:5173`

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | FastAPI (Python, async) |
| Frontend | React + Vite + Tailwind CSS v3 |
| Charts | Recharts (PieChart donut, AreaChart) |
| Price / company data | Massive API (Polygon.io-compatible REST) |
| Analyst data | yfinance (free, no API key) |

## Data Sources

**Massive API** (`api.massive.com`) — API key in `.env`
- Company reference: `/v3/reference/tickers/{symbol}` — name, description, market cap, employees, branding
- Current price: `/v2/aggs/ticker/{symbol}/range/1/day/{from}/{to}` — last 2 bars to get close + prior close
- 90-day chart: same endpoint, ascending sort, 95-bar limit

**yfinance** (`yf.Ticker(symbol)`)
- `analyst_price_targets` → low / mean / high / median / numberOfAnalysts
- `recommendations` → strong_buy / buy / hold / sell / strong_sell counts by period
- `info` → recommendationKey (buy/hold/sell), recommendationMean
- `upgrades_downgrades` → last 20 rating changes with firm, action, from/to grade

> Note: Massive API's Benzinga analyst endpoints (`/benzinga/v1/ratings`) return 403 on the current plan. yfinance covers this gap for free.

## Architecture

```
Browser → Vite dev server (:5173)
              │  proxy /api/* 
              ▼
         FastAPI (:8000)
              ├── GET /api/ticker/{symbol}        → Massive API (reference + price)
              ├── GET /api/ticker/{symbol}/analyst → yfinance
              └── GET /api/ticker/{symbol}/chart   → Massive API (OHLC bars)
```

All three endpoints are called in parallel (`Promise.all`) on search.

## Components

| Component | Description |
|---|---|
| `CompanyHeader` | Logo, name, exchange, BUY/HOLD/SELL badge, price + change, market cap, employees, description |
| `AnalystRatingCard` | Donut chart (Recharts PieChart), legend with counts and %, "X% bullish" headline |
| `PriceTargetCard` | CSS range bar: low → high, dot markers for current price and mean target, upside % |
| `PriceChart` | 90-day area chart, green/red based on direction, custom tooltip |
| `UpgradesTable` | Last 20 rating changes — date, firm, action badge (color-coded), from/to grade |

## Running Locally

```bash
# Backend
cd ~/src/ticker-research/backend
uvicorn main:app --host 0.0.0.0 --port 8000

# Frontend (separate terminal)
cd ~/src/ticker-research/frontend
npm run dev -- --host 0.0.0.0 --port 5173
```

The Vite dev server proxies `/api/*` to `localhost:8000`, so it works from any hostname (localhost or Tailscale).

## Code Location

- Repo: `~/src/home_infra` — branch `feat/ticker-research`
- Path: `ticker-research/backend/` and `ticker-research/frontend/`
- `.env` files (gitignored): contain `MASSIVE_API_KEY`

## Next Steps

- [ ] Deploy to k3s on melody-beast (Kubernetes manifests, Dockerfile for backend, static build for frontend)
- [ ] Add options chain view — expirations, strikes, IV, bid/ask
- [ ] Add earnings calendar
- [ ] Wire into the broader options trading workflow
