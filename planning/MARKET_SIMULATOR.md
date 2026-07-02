# Market Simulator Design — GBM-Based Price Simulation

## 1. Goal

Per `PLAN.md` §6, when `MASSIVE_API_KEY` is not set the app must still feel alive: 10+ tickers streaming plausible, correlated, continuously-moving prices with occasional dramatic moves — with zero external dependencies, so the whole app runs offline out of the box. This document designs the simulator that produces that experience.

## 2. Why Geometric Brownian Motion

Real stock prices are commonly modeled as GBM: returns are log-normally distributed, prices can't go negative, and volatility scales with the price level rather than being a fixed dollar amount. GBM is the standard choice for this kind of illustrative simulation because it's cheap to compute per tick, well understood, and produces visually realistic-looking random walks without needing a real historical dataset to sample from.

**The GBM update rule:**

```
S(t+dt) = S(t) * exp( (mu - sigma^2/2) * dt + sigma * sqrt(dt) * Z )
```

Where:
- `S(t)` — current price
- `mu` — annualized drift (expected return)
- `sigma` — annualized volatility
- `dt` — time step, expressed as a fraction of a trading year
- `Z` — a standard normal random draw (correlated across tickers — see §4)

The `- sigma^2/2` term is the standard Itô correction that keeps the *expected* price growth equal to `mu` despite the multiplicative noise (without it, volatility alone would bias the price upward over time).

## 3. Choosing the Time Step

Prices update every 500ms (`PLAN.md` §6). `dt` must be expressed as a fraction of a trading year so that `mu`/`sigma` (annualized figures anyone can sanity-check, e.g. "22% annual volatility") produce the right *magnitude* of per-tick movement.

```
TRADING_SECONDS_PER_YEAR = 252 trading days * 6.5 hours/day * 3600 s/hour = 5,896,800
DEFAULT_DT = 0.5 seconds / 5,896,800 ≈ 8.48e-8
```

This tiny `dt` is intentional: it produces sub-cent moves on any single tick, which accumulate into a realistic-looking multi-cent random walk over the course of seconds and minutes — rather than a jumpy, unrealistically volatile display if `dt` were miscalibrated.

## 4. Correlated Moves Across Tickers

Real markets don't move independently — tech stocks tend to move together, financials move together, and some names (famously, high-beta/idiosyncratic names) are more independent. `PLAN.md` §6 explicitly calls for "correlated moves across tickers (e.g., tech stocks move together)."

**Approach: Cholesky decomposition of a correlation matrix.**

1. Build an `n x n` correlation matrix where `n` = number of active tickers, based on sector grouping.
2. Compute its Cholesky decomposition `L` (`corr = L @ L.T`).
3. Each tick, draw `n` independent standard normals `Z_independent`, then transform: `Z_correlated = L @ Z_independent`.
4. Feed `Z_correlated[i]` into ticker `i`'s GBM step instead of an independent draw.

This is the standard technique for generating correlated multivariate normal samples from independent ones, and it's `O(n^2)` to rebuild (fine for `n < 50` tickers) and `O(n^2)` per tick to apply (a single matrix-vector multiply), so it's cheap enough for the 500ms hot path.

**Correlation structure (sector-based grouping):**

| Group pairing | Correlation | Rationale |
|---|---|---|
| Two tech tickers (AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX) | 0.6 | Tech names commonly co-move on macro/sector news |
| Two finance tickers (JPM, V) | 0.5 | Financials co-move on rate/economic news |
| Either ticker is TSLA | 0.3 | TSLA is deliberately modeled as more idiosyncratic — "does its own thing" |
| Cross-sector, or either ticker unknown (dynamically added) | 0.3 | Baseline market-wide co-movement, no strong sector link assumed |

The correlation matrix (and its Cholesky factor) is rebuilt whenever a ticker is added or removed via the watchlist, since `n` and the group membership change.

## 5. Seed Prices & Per-Ticker Parameters

Each of the 10 default tickers gets a realistic starting price and its own `(mu, sigma)` pair, rather than one global volatility for every name — this is what makes NVDA feel like a volatile momentum name and JPM feel like a steady blue-chip in the UI.

| Ticker | Seed price | sigma (annualized vol) | mu (annualized drift) | Notes |
|---|---|---|---|---|
| AAPL | $190.00 | 0.22 | 0.05 | |
| GOOGL | $175.00 | 0.25 | 0.05 | |
| MSFT | $420.00 | 0.20 | 0.05 | |
| AMZN | $185.00 | 0.28 | 0.05 | |
| TSLA | $250.00 | 0.50 | 0.03 | High volatility |
| NVDA | $800.00 | 0.40 | 0.08 | High volatility, strong drift |
| META | $500.00 | 0.30 | 0.05 | |
| JPM | $195.00 | 0.18 | 0.04 | Low volatility (bank) |
| V | $280.00 | 0.17 | 0.04 | Low volatility (payments) |
| NFLX | $600.00 | 0.35 | 0.05 | |

Tickers added dynamically at runtime (via the AI chat or manual watchlist add) that aren't in this table fall back to a default `(mu=0.05, sigma=0.25)` and a random seed price in `[$50, $300]` — plausible enough for an arbitrary unrecognized symbol without special-casing it.

## 6. Random "Events" for Drama

`PLAN.md` §6 calls for "occasional random events — sudden 2-5% moves on a ticker for drama." Pure GBM alone looks smooth and a little dull over a short demo session; without occasional discrete jumps, users watching for a minute or two might not see anything eventful.

**Mechanism:** each tick, independently per ticker, roll a `0.1%` (`event_probability=0.001`) chance of a shock. If triggered, multiply the price by `1 ± U(0.02, 0.05)` (a uniformly random 2–5% jump, sign chosen at random). With ~10 tickers at 2 ticks/second, this produces roughly one visible "spike" event across the whole watchlist every ~50 seconds on average — frequent enough to notice, rare enough not to feel noisy or gimmicky.

## 7. Code Structure

```python
class GBMSimulator:
    """Pure simulation math — no async, no I/O, no cache dependency."""

    def __init__(self, tickers, dt=DEFAULT_DT, event_probability=0.001): ...
    def step(self) -> dict[str, float]: ...       # advance all tickers one tick, return new prices
    def add_ticker(self, ticker) -> None: ...       # rebuilds Cholesky
    def remove_ticker(self, ticker) -> None: ...    # rebuilds Cholesky
    def get_price(self, ticker) -> float | None: ...
    def get_tickers(self) -> list[str]: ...


class SimulatorDataSource(MarketDataSource):
    """Async wrapper: owns the loop, writes GBMSimulator output into the shared PriceCache."""

    def __init__(self, price_cache, update_interval=0.5, event_probability=0.001): ...
    async def start(self, tickers) -> None: ...     # seeds cache immediately, launches loop
    async def stop(self) -> None: ...
    async def add_ticker(self, ticker) -> None: ...   # delegates to GBMSimulator, seeds cache
    async def remove_ticker(self, ticker) -> None: ... # delegates + evicts from cache
    def get_tickers(self) -> list[str]: ...

    async def _run_loop(self) -> None:
        while True:
            prices = self._sim.step()
            for ticker, price in prices.items():
                self._cache.update(ticker=ticker, price=price)
            await asyncio.sleep(self._interval)
```

**Why split pure math (`GBMSimulator`) from the async wrapper (`SimulatorDataSource`)?**
- `GBMSimulator.step()` is deterministic-shaped, synchronous, and has no dependency on asyncio, the cache, or logging — it's trivial to unit test directly (seed the RNG, call `step()` N times, assert prices stay positive and drift/volatility are in the right ballpark) without spinning up an event loop.
- `SimulatorDataSource` is the only piece that implements the shared `MarketDataSource` contract (see `MARKET_INTERFACE.md` §4) and is the only piece that touches `PriceCache` or `asyncio.sleep`. Keeping it thin means the interesting math is easy to test in isolation, and the interface-conformance is easy to verify separately.

**Hot path performance:** `step()` runs every 500ms regardless of ticker count (currently ≤ ~20 in practice). All per-tick work is vectorized with `numpy` (batch draw of `n` normals, one matrix-vector multiply for correlation, then a per-ticker Python loop for the GBM formula itself) — comfortably fast enough that simulation cost is negligible next to the 500ms cadence.

## 8. What This Deliberately Does *Not* Model

- No order book, bid/ask spread, or liquidity effects — out of scope per `PLAN.md`'s "market orders only" simplification.
- No true intraday seasonality (e.g., higher volatility at market open/close) — constant `sigma` per ticker is enough for a convincing demo without adding a time-of-day model.
- No mean reversion or regime switching — pure GBM plus occasional independent shocks is sufficient "drama" without the complexity of a jump-diffusion or stochastic-volatility model.

These are conscious simplicity trade-offs, not oversights — if the simulator ever needs to feel more realistic, the correlation and event-shock mechanisms are the two places to extend first.
