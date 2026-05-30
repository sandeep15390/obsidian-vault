# Market Time Machine

## Overview

An options backtesting and replay tool. The long-term goal is to simulate full options strategies through historical time — entering positions, stepping forward day by day, and watching P&L and Greeks evolve. The MVP narrows scope to visualization: see how a specific ORCL options contract price evolved alongside the underlying stock price, so you can study patterns before building strategy simulation.

## MVP Scope (current)

- **Single stock:** ORCL (Oracle) only
- **Data source:** Alpaca paper trading API — historical options bars and ORCL stock bars for the past 6 months (data available from February 2024 onward)
- **Storage:** Local SQLite database — no server required
- **Backend:** FastAPI serving contract list and price series from SQLite
- **Frontend:** Streamlit single-page app — date range picker, contract selector, Plotly dual-axis chart
- **Chart:** ORCL stock price (left axis) overlaid with one or more selected option contract prices (right axis); toggle between "daily snapshot" and "full timeline to expiry" views
- **Observability:** Structured JSON logging (structlog), Prometheus metrics endpoint on the FastAPI backend

See [[Architecture]] for full component design, data model, API spec, test plan, and implementation phases.

## Long-Term Goals

- Simulate options strategies using historical market data (spreads, straddles, iron condors)
- Let users "live through" a trade by stepping forward in time rather than just seeing a final P&L
- Real-time P&L, Greeks, and scenario visualization as time progresses
- Replay mode to feel the emotional arc of a trade
- Multi-stock support

## Pages

- [[Architecture]] — Full MVP architecture: data model, API, frontend, observability, tests, implementation plan

## Ideas & Notes

- Greeks (delta, theta, IV) are not available in historical bars from Alpaca — only in live snapshots. Post-MVP: run a daily cron to capture live snapshots and accumulate Greeks history.
- Alpaca historical options data starts February 2024 — the MVP 6-month window falls within this range.
- The `indicative` (free) feed is sufficient for historical visualization. `opra` feed (paid) would be needed for real-time strategy automation.

## Resources

- Alpaca paper trading account: `app.alpaca.markets` → Paper Trading → API Keys
- Alpaca options API docs: https://docs.alpaca.markets/docs/historical-option-data
- alpaca-py SDK: https://alpaca.markets/sdks/python/

## Status

- [ ] Verify Alpaca ORCL option chain data availability (run Q1 probe command from Architecture doc)
- [ ] Verify `get_option_bars` returns data for expired contracts (Q2 probe)
- [ ] Phase 1: repo scaffold, virtualenv, `.env` wiring, `setup.sh`
- [ ] Phase 2: data ingestion (`ingest.py`)
- [ ] Phase 3: FastAPI backend
- [ ] Phase 4: Streamlit frontend
- [ ] Phase 5: observability (structlog + prometheus-client)
- [ ] Phase 6: test suite passing
- [ ] Phase 7: teardown/reinstall validation (3 destructive cycles)
