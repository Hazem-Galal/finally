# Market Interface Design — Unified Market Data API

## 1. Goal

The rest of the backend (SSE streaming, portfolio valuation, trade execution) should never know or care whether prices come from the real Massive API or from an in-process simulator. `PLAN.md` §5/§6 specifies the switch is driven purely by whether `MASSIVE_API_KEY` is set. This document designs the abstraction that makes that possible: one `MarketDataSource` interface, one shared `PriceCache`, and a factory that picks the implementation at startup.

```
MASSIVE_API_KEY set & non-empty ──▶ MassiveDataSource ──┐
                                                          ├──▶ PriceCache ──▶ SSE stream / portfolio / trades
MASSIVE_API_KEY unset/empty ─────▶ SimulatorDataSource ──┘
```

Everything downstream of `PriceCache` is agnostic to the source — this also means a future third source (e.g., a different vendor) only needs to implement the same interface.

## 2. Package Layout

```
backend/app/market/
├── __init__.py       # Public exports
├── models.py          # PriceUpdate dataclass
├── cache.py           # PriceCache (thread-safe shared store)
├── interface.py        # MarketDataSource abstract base class
├── massive_client.py   # MassiveDataSource (real data via Massive REST API)
├── simulator.py         # GBMSimulator + SimulatorDataSource
├── seed_prices.py       # Seed prices / volatility / correlation params
├── factory.py           # create_market_data_source(cache) -> MarketDataSource
└── stream.py            # SSE router reading from PriceCache
```

## 3. Core Data Model — `PriceUpdate`

An immutable snapshot of one ticker's price at a point in time. Both data sources produce these (indirectly, via `PriceCache.update()`), so the shape is fixed regardless of source.

```python
@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float: ...          # price - previous_price
    @property
    def change_percent(self) -> float: ...   # % change, guarded against div-by-zero
    @property
    def direction(self) -> str: ...          # "up" | "down" | "flat"

    def to_dict(self) -> dict: ...            # JSON-serializable, used directly by SSE
```

Design choices:
- **Frozen + slots**: cheap to construct at high frequency (every ticker, every tick) with no accidental mutation once published.
- **Derived properties, not stored fields**: `change`/`change_percent`/`direction` are computed from `price`/`previous_price` rather than passed in, so there's exactly one place that can get the math wrong.
- **`to_dict()` lives on the model**: the SSE layer and any future REST endpoint serialize the same way without duplicating field lists.

## 4. The Abstract Contract — `MarketDataSource`

```python
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

**Lifecycle contract:**

```python
source = create_market_data_source(cache)
await source.start(["AAPL", "GOOGL", ...])   # call exactly once
...
await source.add_ticker("TSLA")               # watchlist add, any time after start()
await source.remove_ticker("GOOGL")           # watchlist remove, any time after start()
...
await source.stop()                            # idempotent, safe to call multiple times
```

Why this exact shape:

| Method | Rationale |
|---|---|
| `start(tickers)` | Takes the initial ticker list (from the DB-backed watchlist) so the source can seed itself and begin its background loop in one step, rather than requiring a separate "add all" dance. |
| `stop()` | Explicitly separate from `__del__`/GC — background tasks (asyncio tasks, poll loops) need deterministic, awaitable shutdown, e.g. on FastAPI app shutdown. |
| `add_ticker` / `remove_ticker` | Mirror watchlist mutations 1:1, so the API layer that handles `POST /api/watchlist` and `DELETE /api/watchlist/{ticker}` can call these directly without knowing which source is active. |
| `get_tickers()` | Synchronous introspection (e.g., for debugging/health checks) — no need to await just to see what's tracked. |
| All mutation methods `async` | Both real implementations eventually touch shared async state (a poll list, an asyncio task); making the contract async from day one avoids a later breaking change if the simulator's internals ever need to await something. |

Both concrete sources — `MassiveDataSource` and `SimulatorDataSource` — implement this exact interface and are otherwise unrelated in their internals.

## 5. The Shared Sink — `PriceCache`

```python
class PriceCache:
    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate: ...
    def get(self, ticker: str) -> PriceUpdate | None: ...
    def get_all(self) -> dict[str, PriceUpdate]: ...
    def get_price(self, ticker: str) -> float | None: ...
    def remove(self, ticker: str) -> None: ...
    @property
    def version(self) -> int: ...   # monotonically increasing, bumped on every update
```

- **One writer at a time, many readers.** Exactly one `MarketDataSource` background task writes to a given cache instance; the SSE stream, portfolio valuation, and trade execution all read from it concurrently. A plain `threading.Lock` around the internal dict is sufficient — updates are frequent (every ~500ms per ticker) but cheap, and reads are point lookups or full-dict snapshots.
- **`update()` computes `previous_price` internally** by looking at whatever was cached before the call. Callers (both data sources) only ever provide the new price — this guarantees `PriceUpdate.change`/`direction` are always derived consistently regardless of which source produced the update.
- **`version` counter enables cheap SSE polling.** Rather than diffing price dicts on every tick, the SSE generator just compares `cache.version` to the last value it saw and skips sending a frame if nothing changed (see §7).

## 6. Selecting an Implementation — `factory.py`

```python
def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()
    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    return SimulatorDataSource(price_cache=price_cache)
```

This is the entire policy decision for the whole application, in one function. Notes:
- `.strip()` guards against an accidentally-set-but-blank env var (e.g. `MASSIVE_API_KEY=` in a `.env` file) silently trying to hit the real API with an empty key.
- The factory returns an **unstarted** source — the caller (app startup) is responsible for calling `await source.start(tickers)` with the watchlist loaded from the database. This keeps the factory free of any I/O or async context requirements, so it's trivial to call from synchronous setup code and to unit test.

## 7. How Downstream Code Uses This

**SSE streaming (`stream.py`)** reads only from `PriceCache`, never from a `MarketDataSource` directly:

```python
current_version = price_cache.version
if current_version != last_version:
    last_version = current_version
    prices = price_cache.get_all()
    yield f"data: {json.dumps({t: u.to_dict() for t, u in prices.items()})}\n\n"
```

**Watchlist API routes** call the source's `add_ticker`/`remove_ticker`, obtained once at startup and stored on `app.state`:

```python
@router.post("/api/watchlist")
async def add_to_watchlist(body: AddTickerRequest, request: Request):
    await request.app.state.market_source.add_ticker(body.ticker)
    # ... persist to DB ...
```

**Portfolio valuation / trade execution** call `price_cache.get_price(ticker)` for the current fill price — they never construct or reference a `MarketDataSource` at all.

This three-layer separation (source → cache → consumers) is what makes the Massive/simulator swap invisible to 95% of the codebase: only `factory.py` and app startup know the concrete type in play.

## 8. Concrete Implementation Notes

### 8.1 `SimulatorDataSource`

Wraps a `GBMSimulator` (see `MARKET_SIMULATOR.md`) and an `asyncio` loop that calls `.step()` every `update_interval` seconds (default 0.5s), writing each resulting price into the cache. `add_ticker`/`remove_ticker` mutate the simulator's internal state directly (in-process, no I/O, so these return immediately).

### 8.2 `MassiveDataSource`

Wraps the synchronous `massive.RESTClient` and an `asyncio` loop that polls the batched snapshot endpoint every `poll_interval` seconds (default 15.0s, matched to the free tier's 5 req/min — see `MASSIVE_API.md` §4). Because the SDK is synchronous, each poll runs via `asyncio.to_thread(...)` so it doesn't block the event loop or the SSE generators sharing it. `add_ticker`/`remove_ticker` just mutate the in-memory ticker list — the change takes effect on the *next* poll cycle rather than immediately, which is an acceptable latency trade-off given the 15s cadence.

Both implementations independently handle their own failure modes (simulator: none expected, it's pure math; Massive: log-and-continue on `401`/`429`/network errors so a transient API hiccup never crashes the background task or the app).

## 9. Testing Implications

Because both sources share one interface, the test suite can:
- Run the exact same `MarketDataSource` contract tests (start/stop/add/remove/get_tickers behavior) against both implementations.
- Test the SSE stream and portfolio code entirely against a `PriceCache` populated by hand, with no dependency on either real source — the cache is the seam.
- Mock only `RESTClient`/`massive` at the boundary of `MassiveDataSource`, keeping Massive-specific test complexity isolated to `test_massive.py`.
