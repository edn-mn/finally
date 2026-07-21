# Unified Market Data Interface — Design

Design doc for the Python abstraction that lets FinAlly get live-ish stock prices from
either the Massive API or an in-process simulator, selected by whether `MASSIVE_API_KEY`
is set. Written from fresh research against the current Massive SDK (see `MASSIVE.md`),
then checked against the existing implementation in `backend/app/market/` (built earlier,
documented as complete in `MARKET_DATA_SUMMARY.md`). This doc supersedes
`planning/archive/MARKET_INTERFACE.md`, which predates the Polygon.io -> Massive rebrand
and the field-name corrections below.

## Bottom line

**The design is right; one line of the implementation is wrong.** The strategy-pattern
abstraction (`MarketDataSource` ABC, `PriceCache` as the single point of truth, factory
selecting on an env var) is the correct shape and needs no redesign. But
`MassiveDataSource._poll_once` reads an attribute that doesn't exist on the real SDK
response, which — running against a real Massive account rather than the test suite's
mocks — means it would silently fail to update any price, every poll, forever. See
[The bug](#the-bug-lasttradetimestamp-does-not-exist).

## The interface

```python
from abc import ABC, abstractmethod

class MarketDataSource(ABC):
    @abstractmethod
    async def start(self, tickers: list[str]) -> None: ...

    @abstractmethod
    async def stop(self) -> None: ...

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None: ...

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None: ...

    @abstractmethod
    def get_tickers(self) -> list[str]: ...
```

This is exactly what `backend/app/market/interface.py` already has. Both
implementations write into a shared `PriceCache`; nothing downstream (SSE stream,
trade execution, portfolio valuation) needs to know which source is active. That
agnosticism is the entire point of the interface and it holds up under the fresh
research — nothing about the real Massive API demands a different shape.

## Source selection

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    return SimulatorDataSource(price_cache=price_cache)
```

Matches `backend/app/market/factory.py` verbatim. One thing worth naming explicitly in
this doc that wasn't obvious before the fresh research: **`MASSIVE_API_KEY` is not just
our env var name — it's the Massive SDK's own default env var** (`ENV_KEY =
"MASSIVE_API_KEY"` in `massive/rest/__init__.py`; `RESTClient()` reads it automatically).
The env-var-driven design in `PLAN.md` §5 lines up with the SDK for free — `RESTClient()`
with no arguments already does the right thing once the env var is set. The existing code
passes `api_key=` explicitly instead of relying on that default, which is fine (it's
resolved once already), just noting it isn't required.

## Why the two sources can share one cache shape

`PriceUpdate` (in `models.py`) only needs `ticker, price, previous_price, timestamp`, and
derives `change`, `change_percent`, `direction` from those four fields. Neither data
source is asked to supply a pre-computed day-change — `PriceCache.update()` computes
`previous_price` itself from whatever was cached before the current write:

```python
prev = self._prices.get(ticker)
previous_price = prev.price if prev else price
```

This is a deliberate simplification worth calling out because the real Massive snapshot
*does* carry a genuine previous-close (`snap.prev_day.close`) and a genuine day-change
(`snap.todays_change_percent`) — fields that would let the Massive-backed source report
"change since market open" rather than "change since last poll." The current code
ignores both and computes poll-to-poll change instead, same as the simulator does tick-
to-tick. That's the right call for this project: it keeps the two sources byte-for-byte
consistent (a frontend flash animation or sparkline can't tell which source is behind
the price), and "change since last poll" is indistinguishable from "day change" to a
user watching a live demo. Don't wire up `prev_day`/`todays_change_percent` unless a
future requirement specifically needs true intraday-open-to-now change — it would only
reintroduce a divergence between the two sources for no visible benefit.

## The Massive-backed source

```python
class MassiveDataSource(MarketDataSource):
    async def start(self, tickers): ...
    async def _poll_once(self):
        snapshots = await asyncio.to_thread(self._fetch_snapshots)
        for snap in snapshots:
            price = snap.last_trade.price
            timestamp = snap.last_trade.timestamp / 1000.0
            self._cache.update(ticker=snap.ticker, price=price, timestamp=timestamp)

    def _fetch_snapshots(self):
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS, tickers=self._tickers,
        )
```

(condensed from `backend/app/market/massive_client.py`)

Design-wise this is right: one batched `get_snapshot_all()` call per poll cycle keeps a
10-ticker watchlist well under the free tier's 5 req/min at a 15s interval, the
synchronous SDK call is offloaded via `asyncio.to_thread` so it doesn't block the event
loop, and per-ticker failures are caught individually so one bad snapshot doesn't kill
the whole poll.

### The bug: `last_trade.timestamp` does not exist

Verified directly against `massive/rest/models/trades.py` in the current SDK source:

```python
class LastTrade:
    ticker: Optional[str] = None
    trf_timestamp: Optional[int] = None
    sequence_number: Optional[float] = None
    sip_timestamp: Optional[int] = None
    participant_timestamp: Optional[int] = None
    ...
    price: Optional[float] = None
    ...
```

There is no `timestamp` attribute — only `sip_timestamp`, `participant_timestamp`,
`trf_timestamp`. `massive_client.py:103` does:

```python
timestamp = snap.last_trade.timestamp / 1000.0
```

This raises `AttributeError` on every real snapshot. `_poll_once` does catch
`(AttributeError, TypeError)` per-ticker and logs a warning, so the process won't crash
— but the practical effect is that **every ticker fails on every poll**, so the price
cache would never receive an update from a real Massive-backed run. The watchlist would
show stale seed-less/blank prices indefinitely whenever `MASSIVE_API_KEY` is set.

Why the existing test suite didn't catch this: `tests/market/test_massive.py` mocks
`snap.last_trade` with `MagicMock()` and does `snap.last_trade.timestamp = timestamp_ms`
— `MagicMock` happily accepts an assignment to any attribute name, including one that
doesn't exist on the real dataclass. 73 passing tests and 84% coverage validate the
mock's shape, not the SDK's actual shape. This is a mock/reality drift, not a logic bug
in `PriceCache` or the polling loop itself.

**Fix** (two changes, both in `_poll_once`):
```python
price = snap.last_trade.price
timestamp = snap.last_trade.sip_timestamp / 1_000_000_000  # nanoseconds, not ms
```
`sip_timestamp` is the SIP-consolidated trade timestamp in Unix **nanoseconds** (per the
SDK's own `LastTrade.from_dict`, which maps JSON key `"t"` to `sip_timestamp`). The
archived doc's claim that Massive timestamps are "Unix milliseconds" is also stale —
that was true of an older response shape; the current field is nanoseconds.

This is a one-line-plus-divisor fix and a corresponding test-mock fix (mock
`sip_timestamp`, not `timestamp`), not a design change. Flagging it here rather than
patching it, since this task was scoped to research and documentation — it's an
actionable item for whoever next touches `massive_client.py`.

## The simulator-backed source

No corrections needed here — this doesn't depend on Massive at all, and the design
already matches `PLAN.md` §6 closely: GBM per ticker, Cholesky-correlated draws by
sector group, ~0.1%/tick chance of a 2-5% shock event, seeded from `SEED_PRICES` with a
`random.uniform(50, 300)` fallback for tickers with no predefined seed. One byte-level
note: `PLAN.md` describes the simulator's dynamic-ticker fallback as "a randomized
starting price in a plausible range" — worth optionally grounding that in
`get_previous_close_agg()` (see `MASSIVE.md`) if a future iteration wants newly-added
tickers to start near their real price instead of a $50-300 dice roll, but that's a
nice-to-have, not a gap.

## SSE layer

`create_stream_router` / `_generate_events` in `stream.py` already do exactly what
`PLAN.md` §6 specifies: one JSON object per tick keyed by ticker, version-counter-gated
so no event is sent when nothing changed, `retry: 1000` for EventSource auto-reconnect.
Nothing about the fresh Massive research changes this layer — it's already fully
source-agnostic, reading only from `PriceCache`.

## Reconciliation summary

| Area | Fresh research says | Current code | Verdict |
|---|---|---|---|
| Env var name | SDK default is `MASSIVE_API_KEY` | Uses `MASSIVE_API_KEY` | Matches, no change |
| Batch snapshot call | `get_snapshot_all(market_type=..., tickers=[...])` | Same | Matches, no change |
| Price field | `snap.last_trade.price` | Same | Matches, no change |
| Timestamp field | `snap.last_trade.sip_timestamp` (nanoseconds) | `snap.last_trade.timestamp` (doesn't exist) | **Bug — fix needed** |
| Previous close / day change | `snap.prev_day.close` / `snap.todays_change_percent` available | Deliberately unused; computed in `PriceCache` instead | Matches intent, no change |
| WebSocket streaming | Requires Advanced plan ($199/mo), only real-time on Developer+ | Not used; REST polling only | Matches, confirms the original choice was right |
| Rate limiting | Free tier 5 req/min, SDK retries 429/5xx automatically | 15s poll default | Matches, no change |
