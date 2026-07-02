# Massive API Research — Real-Time & End-of-Day Prices for Multiple Tickers

## 1. What Is Massive

[Massive](https://massive.com) is the rebranded identity of **Polygon.io** (rebrand completed October 30, 2025). It is a financial market data API covering stocks, options, indices, forex, and crypto. Existing Polygon.io API keys, accounts, and integrations continue to work unchanged — the client library defaults to `api.massive.com` but remains backward compatible with `api.polygon.io` and existing keys.

For this project we only need the **Stocks REST API**: multi-ticker snapshots (near-real-time) and previous-day/aggregate bars (end-of-day).

Sources:
- [Massive Docs](https://massive.com/docs)
- [Stocks REST API Overview](https://massive.com/docs/rest/stocks/overview)
- [Full Market Snapshot](https://massive.com/docs/rest/stocks/snapshots/full-market-snapshot)
- [Previous Day Bar (OHLC)](https://massive.com/docs/rest/stocks/aggregates/previous-day-bar)
- [Official Python client](https://github.com/massive-com/client-python)
- [Massive + Python blog post](https://massive.com/blog/polygon-io-with-python-for-stock-market-data)
- [Request limit FAQ](https://massive.com/knowledge-base/article/what-is-the-request-limit-for-massives-restful-apis)

## 2. Authentication

Every REST call requires an API key, obtained from the [Massive dashboard](https://massive.com/dashboard/api-keys). The key is passed either as the `apiKey` query parameter on raw HTTP calls, or once to the `RESTClient` constructor when using the official Python SDK — the SDK attaches it to every request automatically.

```python
from massive import RESTClient

client = RESTClient(api_key="YOUR_MASSIVE_API_KEY")
```

## 3. Installation

```bash
pip install -U massive     # or: uv add massive
```

Requires Python 3.9+. The package name is `massive` (the old `polygon-api-client` package is superseded but still installable for legacy code).

## 4. Rate Limits & Tiers

| Tier | Requests | Notes |
|---|---|---|
| Free | **5 requests/minute** | Suitable for polling a small watchlist every ~15s |
| Paid (Starter and up) | Effectively unlimited requests | Real-time or 15-min-delayed data depending on plan; polling can drop to 2–5s |

Implication for this project: with the free tier and a single API call per poll cycle (batched snapshot request for all watched tickers), a 15-second poll interval stays safely under 5 req/min. This is why the project's `MassiveDataSource` defaults to `poll_interval=15.0`.

## 5. Key Endpoints

### 5.1 Multi-Ticker Snapshot (near-real-time)

```
GET /v2/snapshot/locale/us/markets/stocks/tickers
```

Returns the latest trade, latest quote, most recent minute bar, today's aggregate bar, and the previous day's bar — for many tickers in **one request**. This is the endpoint used to drive the live watchlist.

**Query parameters:**

| Param | Type | Required | Description |
|---|---|---|---|
| `tickers` | string | No | Case-sensitive comma-separated list, e.g. `AAPL,MSFT,TSLA`. Omit to get the entire market (10,000+ tickers) — always pass an explicit list for this project. |
| `include_otc` | boolean | No | Include OTC securities. Default `false`. |

**Response shape (abridged):**

```json
{
  "status": "OK",
  "count": 3,
  "tickers": [
    {
      "ticker": "AAPL",
      "todaysChange": 1.23,
      "todaysChangePerc": 0.65,
      "updated": 1735689600000,
      "day": { "o": 189.5, "h": 191.2, "l": 188.9, "c": 190.4, "v": 42000000, "vw": 190.1 },
      "prevDay": { "o": 187.0, "h": 190.0, "l": 186.5, "c": 189.17, "v": 51000000, "vw": 188.4 },
      "lastTrade": { "p": 190.42, "s": 100, "t": 1735689599000 },
      "lastQuote": { "p": 190.40, "P": 190.44, "s": 2, "S": 4, "t": 1735689599500 },
      "min": { "o": 190.3, "h": 190.5, "l": 190.2, "c": 190.4, "v": 15000, "t": 1735689540000 }
    }
  ]
}
```

Field notes relevant to this project:
- `lastTrade.p` — the most recent traded price. This is what we treat as "the current price."
- `lastTrade.t` — Unix **milliseconds** timestamp (must divide by 1000 for a Unix-seconds timestamp).
- `prevDay.c` — previous close, useful as a fallback baseline.
- `lastQuote` and `lastTrade` are only populated if the plan tier includes quotes/trades — free/EOD-only plans may only return `day`/`prevDay` bars, which is a graceful degradation path worth handling.
- Snapshot data resets daily around 3:30 AM ET and repopulates as exchanges report, starting as early as 4:00 AM ET.

**Python SDK usage:**

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key="YOUR_KEY")

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "TSLA"],
)

for snap in snapshots:
    print(snap.ticker, snap.last_trade.price, snap.last_trade.timestamp)
```

**Raw HTTP equivalent:**

```bash
curl "https://api.massive.com/v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,GOOGL,MSFT,TSLA&apiKey=YOUR_KEY"
```

### 5.2 Previous Day Bar (end-of-day)

```
GET /v2/aggs/ticker/{stocksTicker}/prev
```

Returns the previous trading day's OHLCV bar for a single ticker. Useful as a baseline/fallback when a live snapshot isn't available (e.g., pre-market, or free-tier EOD-only plans).

**Path parameter:** `stocksTicker` (e.g. `AAPL`)
**Query parameter:** `adjusted` (boolean, default `true`) — adjusts for splits.

**Response shape:**

```json
{
  "ticker": "AAPL",
  "status": "OK",
  "adjusted": true,
  "queryCount": 1,
  "resultsCount": 1,
  "results": [
    { "T": "AAPL", "o": 187.0, "h": 190.0, "l": 186.5, "c": 189.17, "v": 51000000, "vw": 188.4, "t": 1735603200000 }
  ]
}
```

Field key: `T`=ticker, `o`/`h`/`l`/`c`=open/high/low/close, `v`=volume, `vw`=volume-weighted average price, `t`=Unix ms timestamp.

**Python SDK usage:**

```python
prev = client.get_previous_close_agg("AAPL")
bar = prev[0]
print(bar.close, bar.timestamp)
```

Note: there is no batched multi-ticker version of this endpoint — for a watchlist of N tickers this requires N calls. On the free tier (5 req/min) this doesn't scale well, which is one reason the snapshot endpoint (single batched call for all tickers) is preferred for this project even when only EOD-freshness is needed.

### 5.3 Custom Aggregate Bars (historical range, for charting)

```
GET /v2/aggs/ticker/{stocksTicker}/range/{multiplier}/{timespan}/{from}/{to}
```

Returns OHLCV bars over a date/time range at a chosen granularity (`minute`, `hour`, `day`, etc). Not required for the live-price use case but relevant if historical intraday chart backfill is ever added.

```python
aggs = []
for a in client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="minute",
    from_="2026-06-01",
    to="2026-06-30",
    limit=50000,
):
    aggs.append(a)
```

## 6. WebSocket Streaming (considered, not used)

Massive also offers a real-time WebSocket feed (`WebSocketClient`, subscriptions like `T.AAPL` for trades):

```python
from massive import WebSocketClient

ws = WebSocketClient(api_key="YOUR_KEY", subscriptions=["T.AAPL", "T.MSFT"])

def handle_msg(msg):
    for m in msg:
        print(m)

ws.run(handle_msg=handle_msg)
```

**Not used in this project.** Per `PLAN.md` §6/§9, we deliberately use REST polling instead of the WebSocket feed: it works uniformly across all pricing tiers (WebSocket real-time streaming requires a paid plan), and a 15-second REST poll is simple to reason about, easy to rate-limit, and sufficient for a simulated trading terminal.

## 7. Practical Notes for This Project

1. **One batched call per poll cycle.** Always call the snapshot endpoint once with the full comma-separated ticker list rather than once per ticker — this is what keeps the free tier's 5 req/min limit workable.
2. **The `massive` SDK client is synchronous.** In an asyncio application, wrap calls in `asyncio.to_thread(...)` to avoid blocking the event loop (see `MARKET_INTERFACE.md` §4.2).
3. **Timestamps are Unix milliseconds** everywhere in responses — convert to seconds (`/ 1000.0`) before storing, to stay consistent with Python's `time.time()`.
4. **Handle partial/missing fields defensively.** `lastTrade` may be absent depending on plan tier; a ticker with no recent trading activity may be missing from the response entirely. Catch `AttributeError`/`KeyError` per-ticker rather than failing the whole poll cycle.
5. **Failure modes to expect:** `401` (bad/expired key), `429` (rate limit exceeded), transient network errors. None of these should crash the poller — log and retry on the next interval, leaving the last-known price in the cache.
6. **No automatic tier detection.** The SDK doesn't tell you what plan you're on or throttle itself accordingly — poll interval is a manual constructor parameter that the operator must set correctly for their plan (15s default for free tier; can be lowered to 2–5s on paid tiers).
