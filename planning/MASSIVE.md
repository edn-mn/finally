# Massive API Reference (formerly Polygon.io)

Research notes for the Massive REST API, current as of July 2026. Verified directly
against the `massive-com/client-python` source (not just marketing docs), since the
archived `planning/archive/MASSIVE_API.md` from the original build contains a couple of
field-name errors — see [Corrections vs. the archived doc](#corrections-vs-the-archived-doc).

## Overview

- Polygon.io rebranded to **Massive** on 2025-10-30. Existing API keys and accounts
  carried over unchanged.
- **Base URL**: `https://api.massive.com` (the legacy `https://api.polygon.io` host is
  still accepted for a transition period, but the client defaults to the new host).
- **Python package**: `massive` (renamed from `polygon-api-client`).
  Install: `uv add massive` (min Python 3.9).
- **Auth**: `RESTClient()` reads the `MASSIVE_API_KEY` env var automatically — this is
  hardcoded in the SDK (`ENV_KEY = "MASSIVE_API_KEY"` in `massive/rest/__init__.py`), so
  our project's env var name was already the SDK's default, not an arbitrary choice.
  Auth is sent as `Authorization: Bearer <key>`.
- Client defaults: 3 retries on `[413, 429, 499, 500, 502, 503, 504]` with exponential
  backoff, 10s connect/read timeout, pagination on by default.

## Pricing & Data Freshness (important caveat)

| Plan | Price | Data | Notes |
|---|---|---|---|
| Free | $0 | 15-min delayed | 5 requests/minute |
| Stocks Starter | $29/mo | **15-min delayed** | 5yr history |
| Stocks Developer | $79/mo | **Real-time** | 10yr history |
| Stocks Advanced | $199/mo | Real-time, unlimited | Full history, WebSocket streaming |

**"Realtime" REST snapshot data is not actually real-time unless you're on Developer tier
or above.** Free and Starter tiers return data delayed by 15 minutes. For this project
(a course capstone, most likely run on the free tier), assume Massive-backed prices are
delayed — this is fine for a demo but should be documented as a known limitation, not
presented as live streaming.

There is no published hard rate limit beyond the free tier's 5 req/min; paid tiers
recommend staying under 100 req/s.

## Client Initialization

```python
from massive import RESTClient

client = RESTClient()                     # reads MASSIVE_API_KEY from env
client = RESTClient(api_key="explicit_key")  # or pass explicitly
```

## Endpoint: Snapshot — Multiple Tickers (the one we use)

Gets current data for a batch of tickers in **one API call** — this is what makes REST
polling viable for a multi-ticker watchlist under the free tier's 5 req/min limit.

**REST**: `GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT`

**Python**:
```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient()

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],  # comma-list or list, up to 250
)

for snap in snapshots:
    print(f"{snap.ticker}: ${snap.last_trade.price}")
    print(f"  Today's change: {snap.todays_change_percent}%")
    print(f"  Prev close: {snap.prev_day.close}")
```

### `TickerSnapshot` fields (verified from `massive/rest/models/snapshot.py`)

| Attribute | Type | Source JSON key | Notes |
|---|---|---|---|
| `ticker` | str | `ticker` | |
| `day` | `Agg` | `day` | **today's** running bar (open/high/low/close/volume/vwap so far) |
| `prev_day` | `Agg` | `prevDay` | **yesterday's** full daily bar |
| `last_trade` | `LastTrade` | `lastTrade` | most recent trade |
| `last_quote` | `LastQuote` | `lastQuote` | most recent NBBO quote |
| `min` | `MinuteSnapshot` | `min` | most recent minute bar |
| `todays_change` | float | `todaysChange` | absolute $ change since prev close |
| `todays_change_percent` | float | `todaysChangePerc` | % change since prev close |
| `updated` | int | `updated` | Unix **nanoseconds** |

`Agg` (used for both `day` and `prev_day`): `open, high, low, close, volume, vwap,
timestamp, transactions, otc` — note **no `previous_close` field on `Agg`**. To get the
previous close, read `snap.prev_day.close`, not `snap.day.previous_close`.

`LastTrade`: `ticker, price, size, exchange, sip_timestamp` (Unix **nanoseconds**, not
milliseconds — see correction below), plus SIP/exchange metadata we don't need.

### Field-access cheat sheet for our use case

```python
price = snap.last_trade.price
timestamp_seconds = snap.last_trade.sip_timestamp / 1_000_000_000  # ns -> s
previous_close = snap.prev_day.close          # yesterday's close, if needed
day_change_pct = snap.todays_change_percent   # % since yesterday's close
```

## Endpoint: Single Ticker Snapshot

```python
snapshot = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)
print(f"Price: ${snapshot.last_trade.price}")
```

Same `TickerSnapshot` shape as above. Only useful if we ever need a one-off refresh for
a single ticker (e.g. on watchlist add); the batch snapshot already covers the polling
loop.

## Endpoint: Previous Close

```python
prev = client.get_previous_close_agg(ticker="AAPL")
for agg in prev:
    print(f"Previous close: ${agg.close}")
```

**REST**: `GET /v2/aggs/ticker/{ticker}/prev`. Useful for seeding realistic simulator
start prices for a newly-added ticker with no predefined seed (Section 6 of `PLAN.md`),
if we ever want the simulator's placeholder price to be grounded in a real recent price
instead of a randomized range.

## Endpoint: Aggregates (Bars) — for historical charts

Not needed for live polling, but relevant if the frontend's "main chart area" grows a
historical range beyond what's accumulated client-side since page load.

```python
for a in client.list_aggs(
    ticker="AAPL", multiplier=1, timespan="day",
    from_="2026-01-01", to="2026-06-30", limit=50000,
):
    print(a.timestamp, a.open, a.high, a.low, a.close, a.volume)
```

## WebSocket (not used, but considered)

```python
from massive import WebSocketClient

ws = WebSocketClient(subscriptions=["T.AAPL", "T.GOOGL"])  # T. = trades

def handle_msg(msg):
    for m in msg:
        print(m)

ws.run(handle_msg=handle_msg)
```

**Why we don't use this**: real-time WebSocket streaming requires the **Stocks
Advanced plan ($199/mo)** — free and Starter tiers only get 15-minute-delayed data
regardless of transport. Since the project already gates "real data" behind an optional
`MASSIVE_API_KEY` and defaults to a simulator, REST polling on the free tier is the
right fit — a paid-only feature would defeat the "just works with no key" design goal.
This confirms the original architecture decision in `PLAN.md` §3 ("SSE over
WebSockets... simpler, universal") was sound, and extends the reasoning: it's not just
that WebSockets are unnecessary for our one-way push to the browser, it's that Massive's
own WebSocket tier would require a paid plan we don't want to require.

## Error Handling

```python
from massive.exceptions import AuthError, BadResponse

try:
    snapshots = client.get_snapshot_all(market_type=SnapshotMarketType.STOCKS, tickers=tickers)
except AuthError:
    ...  # missing/invalid MASSIVE_API_KEY
except BadResponse:
    ...  # non-200 response (rate limit, server error, etc. after retries exhausted)
```

- `AuthError`: raised client-side if no API key is available at all (env var unset and
  none passed) — this is a `ValueError`-style construction-time error, not an HTTP 401.
  An actual 401 from the server (bad key) surfaces as `BadResponse`.
- `BadResponse`: any non-200 HTTP response, after the client's built-in retry (3
  attempts, exponential backoff) on `429`/`5xx`/`413`/`499` is exhausted.
- During market-closed hours, `last_trade` reflects the last traded price (may be
  after-hours or from the previous session); `day` resets at 12am ET and populates as
  early as 4am ET pre-market.

## Corrections vs. the archived doc

`planning/archive/MASSIVE_API.md` (written during the original market-data build) has
two inaccuracies worth flagging so nobody copies them into new code:

1. **`snap.day.previous_close` does not exist.** The `Agg` model backing `day` has no
   `previous_close` field. Previous close lives on `snap.prev_day.close`.
2. **`snap.day.change_percent` does not exist.** Day-over-day change percent is
   `snap.todays_change_percent` (top-level on `TickerSnapshot`), not nested under `day`.
3. Timestamps: the archived doc says Massive timestamps are "Unix milliseconds" — for
   `last_trade.timestamp` specifically, the current client exposes `sip_timestamp` in
   **nanoseconds**, not milliseconds. (The current backend code already divides by 1000
   assuming milliseconds — see the reconciliation note in `MARKET_INTERFACE.md`.)

None of these affected the actual implementation in `backend/app/market/massive_client.py`,
which only reads `last_trade.price` and `last_trade.timestamp` and computes its own
previous-price/change/direction in `PriceCache` rather than trusting the API's day/prevDay
fields — see `MARKET_INTERFACE.md` for why that turned out to be the right call.
