# Market Time Machine

## Overview
A backtesting simulator for options strategies. Set up a strategy at a point in the past and fast-forward through time to see how it would have played out.

## Goals
- Simulate options strategies using historical market data
- Let users "live through" a trade by stepping forward in time rather than just seeing a final P&L

## Key Features
- Select a historical start date and load market conditions from that point
- Define an options strategy (spreads, straddles, iron condors, etc.)
- Step forward in time (day by day, week by week, or to expiration)
- Real-time P&L, Greeks, and scenario visualization as time progresses
- Replay mode to feel the emotional arc of a trade

## Pages

## Ideas & Notes

## Resources

## Status
- [ ] Identify historical options data source (e.g. CBOE, Polygon.io, Tastytrade API)
- [ ] Define supported strategy types for MVP
- [ ] Design time-travel UI/UX
- [ ] Build options pricing engine (or integrate existing)
- [ ] Build MVP
