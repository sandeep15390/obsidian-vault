---
name: tt
description: >
  Use this skill to interact with the user's Tastytrade account via the `tt`
  CLI. Invoke it when the user asks about their portfolio, positions, balances,
  margin, open orders, order history, options chains, or wants to place/cancel
  a trade (stocks, options, spreads, strangles, iron condors, crypto, futures).
  Trigger phrases: "check my positions", "what's my balance", "show me the
  options chain for X", "sell a put on X", "buy a call spread", "cancel my
  orders", "show order history", "my watchlist", "plot X".
version: 0.1.0
---

# Tastytrade CLI Skill (`tt`)

Use this skill to answer questions about the user's Tastytrade account and to
execute trades on their behalf using the `tt` CLI at `~/.local/bin/tt`.

## Ground Rules

1. **Always confirm before placing or cancelling orders.** Show the exact `tt`
   command you will run and ask the user to confirm before executing it. This
   applies to: `option call`, `option put`, `option strangle`, `trade stock`,
   `trade crypto`, `trade future`, and any cancel action from `order live`.
2. **Read-only commands run immediately** without confirmation: `pf`, `order
   history`, `option chain`, `wl`, `plot`.
3. If the CLI prompts interactively (e.g., to select an expiration or confirm
   an order), relay those prompts verbatim to the user and pass their answer
   back.
4. Never fabricate prices, strikes, or Greeks. Always fetch live data first.

---

## Command Reference

### Portfolio

```
tt pf balance          # cash, net liquidating value, buying power
tt pf positions        # current open positions with P&L
tt pf positions --all  # across all accounts
tt pf margin           # margin usage by position
tt pf history          # closed positions / trade history
```

### Orders

```
tt order live                         # list open/live orders
tt order live --all                   # all accounts
tt order history                      # recent order history
tt order history --symbol AAPL        # filter by symbol
tt order history --start 2026-01-01   # filter by date
tt order history --type "Equity Option"
tt order history --status Filled
```

### Options

```
tt option chain SYMBOL                        # full options chain (monthlies)
tt option chain SYMBOL --weeklies             # include weeklies
tt option chain SYMBOL --dte 30               # expirations near 30 DTE
tt option chain SYMBOL --strikes 5            # N strikes around ATM

tt option call SYMBOL QTY --delta 30          # buy/sell call at ~30 delta
tt option call SYMBOL QTY --strike 150        # buy/sell call at specific strike
tt option call SYMBOL QTY --dte 45            # select expiration by DTE
tt option call SYMBOL QTY --width 5           # turn into a call spread (5-wide)
tt option call SYMBOL QTY --gtc               # GTC instead of day order

tt option put  SYMBOL QTY [same flags as call]
tt option strangle SYMBOL QTY --delta 16      # strangle at ~16 delta each leg
tt option strangle SYMBOL QTY --call 160 --put 140  # specific strikes
tt option strangle SYMBOL QTY --width 5       # iron condor (5-wide wings)
tt option strangle SYMBOL QTY --dte 45 --weeklies
```

Negative `QTY` = sell (open short). Positive `QTY` = buy (open long).

### Stocks / ETFs / Crypto / Futures

```
tt trade stock  SYMBOL QTY     # positive=buy, negative=sell
tt trade stock  SYMBOL QTY --gtc
tt trade crypto SYMBOL QTY
tt trade future SYMBOL QTY
```

### Watchlists

```
tt wl public                    # view public watchlist with prices/metrics
tt wl private                   # view private watchlist
tt wl add    SYMBOL WATCHLIST   # add symbol to private watchlist
tt wl remove SYMBOL WATCHLIST   # remove symbol
tt wl create WATCHLIST          # create new private watchlist
tt wl delete WATCHLIST          # delete private watchlist
```

### Charts

```
tt plot stock  SYMBOL               # 30m candles (default)
tt plot stock  SYMBOL --width 1d    # daily candles
tt plot crypto SYMBOL --width 1h
tt plot future SYMBOL
# Width options: 1m 5m 10m 15m 30m 1h 1d 1mo 1y
```

---

## Workflow

### Read-only requests (run immediately)

1. Parse the user's intent to the correct `tt` subcommand.
2. Run the command with `Bash`.
3. Present the output cleanly — summarize tables, highlight key numbers
   (P&L, delta, DTE, buying power remaining, etc.).

### Trade / order-placement requests

1. **Clarify** any missing parameters: symbol, quantity, strategy type, strike
   or delta target, DTE, spread width, GTC vs day. Ask in one message, not
   one-at-a-time.
2. **Fetch context** if needed: run `tt option chain SYMBOL` or
   `tt pf balance` to ground the conversation in live data before proposing
   a trade.
3. **Propose the command** — show the exact CLI invocation:
   ```
   I'll run:
     tt option put SPY -1 --delta 16 --dte 45
   This sells 1 SPY put at ~16 delta with ~45 DTE. Confirm?
   ```
4. **Wait for explicit user confirmation** (yes/proceed/go ahead). Do not run
   the command until confirmed.
5. **Run the command** and relay the output (order ID, fill price, status).
6. If the CLI asks a follow-up question (expiration selection, final
   confirmation), relay it to the user word-for-word and pass their answer
   back.

### Order cancellation

1. Run `tt order live` to list open orders.
2. Identify the order(s) to cancel based on user description.
3. Confirm: "I'll cancel order #XXXXX for 1 SPY put — OK?"
4. Execute only after confirmation.

---

## Interpreting Output

- **`pf positions`** shows columns like symbol, quantity, avg open price,
  current price, P&L, and delta. Summarize total P&L and flag any large losers.
- **`option chain`** shows strikes, bid/ask, IV, delta, theta, volume, OI.
  When helping the user pick a strike, filter to the relevant expiration and
  highlight the strikes nearest their delta/DTE target.
- **`order live`** shows order ID, symbol, side, quantity, limit price, status.
  Use the order ID when cancelling.
- **`pf balance`** shows cash, net liq, buying power, and margin details.

---

## Common Patterns

**"What's my buying power / balance?"**
→ `tt pf balance`

**"Show my open positions"**
→ `tt pf positions`

**"Show the SPY options chain for 30-45 DTE"**
→ `tt option chain SPY --dte 30 --weeklies`

**"Sell a 16-delta put on SPY, 45 DTE"**
→ Confirm, then: `tt option put SPY -1 --delta 16 --dte 45`

**"Sell a strangle on AAPL"**
→ Ask for delta target and DTE, then propose:
  `tt option strangle AAPL -1 --delta 16 --dte 45 --weeklies`

**"Sell an iron condor on SPX"**
→ Ask for delta and width, then propose:
  `tt option strangle SPX -1 --delta 10 --dte 30 --width 50`

**"What orders do I have open?"**
→ `tt order live`

**"Cancel all my TSLA orders"**
→ `tt order live`, filter for TSLA, confirm each cancellation.

**"Show me a daily chart of QQQ"**
→ `tt plot stock QQQ --width 1d`
