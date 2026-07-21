# Default Watchlist — Ticker Selection

This is a recommendation for which real-world tickers make good **demo/simulator
watchlist entries** for FinAlly — not investment advice. FinAlly trades virtual money
in a simulated portfolio (`PLAN.md` §1-2), so "which stocks to purchase" here means
"which tickers make the app look and feel like a real trading terminal," not a
recommendation about what to do with real money.

## Criteria for a good demo ticker

1. **Plain US common stock on a primary exchange (NYSE/NASDAQ)** — no ADRs, no OTC, no
   dual-class ticker confusion. Keeps both the simulator and the Massive snapshot
   endpoint (`GET /v2/snapshot/locale/us/markets/stocks/tickers`, see `MASSIVE.md`)
   simple — every plan tier covers plain US equities with no extra entitlement needed.
2. **High liquidity / high name recognition** — a course audience should recognize the
   ticker instantly; illiquid small-caps also produce noisier real data that's harder
   to distinguish from simulator noise.
3. **Spread of volatility profiles** — a watchlist where every ticker moves identically
   is a boring demo. Want some low-vol "boring" names and some high-vol "exciting" ones
   so price-flash animations and sparklines actually look different ticker to ticker.
4. **Sector spread that supports correlated simulation** — the GBM simulator groups
   tickers into correlation clusters (`seed_prices.py`: `tech` at ρ=0.6, `finance` at
   ρ=0.5, cross-group at ρ=0.3) so tech names visibly move together and diverge from
   the finance names. Needs at least two distinct sectors represented for that to be
   visible at all.
5. **A volume-2 book covers most of the trades** — the free-tier 5 req/min limit means
   the whole watchlist has to fit in **one** batched snapshot call; that's a ceiling on
   count, not choice of ticker, but it's why 10 (not 50) is the right list length.

## The current default list is a good fit — confirmed per-ticker

| Ticker | Sector | Why it earns a slot |
|---|---|---|
| AAPL | Tech | Most recognizable consumer ticker; moderate volatility (σ≈0.22) |
| GOOGL | Tech | Recognizable, correlates with AAPL/MSFT for the "tech moves together" effect |
| MSFT | Tech | Lowest-vol of the tech cluster (σ≈0.20) — the "boring" tech anchor |
| AMZN | Tech/Retail | Higher vol than MSFT (σ≈0.28), broadens the tech cluster without duplicating it |
| TSLA | Auto/Tech | Highest vol on the list (σ≈0.50) — deliberately decorrelated (ρ=0.3) from the rest, so it visibly "does its own thing" per `seed_prices.py` |
| NVDA | Tech | High vol + strong upward drift (σ≈0.40, μ≈0.08) — most likely to produce a visually satisfying uptrend during a demo |
| META | Tech | Mid-high vol (σ≈0.30), rounds out the tech cluster's spread |
| JPM | Finance | Low vol (σ≈0.18) — anchors the finance cluster, visibly less jumpy than tech |
| V | Finance | Lowest vol on the entire list (σ≈0.17) — pairs with JPM for the finance correlation cluster (ρ=0.5) |
| NFLX | Media/Tech | Second-highest vol (σ≈0.35) — another "exciting" mover distinct from TSLA |

This list already satisfies every criterion above: two clear sectors (7 tech-correlated
names + 2 finance names, with TSLA deliberately independent), a volatility spread from
0.17 (V) to 0.50 (TSLA), all ten are large-cap primary-exchange common stock with no
special API entitlement, and all are instantly recognizable to a course audience.
**No change recommended** — this validates the list already fixed in `PLAN.md` §7 and
`seed_prices.py` rather than proposing a different one.

## If the watchlist is ever expanded

Not a recommendation to act on now (the list is fixed at 10 in the seeded schema), but
if a future iteration wants more tickers, the same criteria point to filling out the
"finance" cluster (currently only 2 names, thinnest of the two groups) before adding a
third sector — e.g. `MA` (pairs with `V`, similar low vol) or `BAC`/`GS` (higher-vol
finance names, would let the finance cluster show internal vol spread the way tech
already does). Adding a third sector (e.g. healthcare or energy) would need a new
`CORRELATION_GROUPS` entry in `seed_prices.py` plus its own pairwise ρ, not just a
`SEED_PRICES`/`TICKER_PARAMS` entry — anything not in a defined group already falls back
to `CROSS_GROUP_CORR` (0.3), so it wouldn't break, it would just render as an
undifferentiated "everything else" bucket rather than its own visible cluster.
