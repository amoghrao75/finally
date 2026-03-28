# Massive API Reference (formerly Polygon.io)

API documentation for retrieving stock market data via the Massive REST API, as used by this project.

## Overview

Polygon.io rebranded as **Massive** on October 30, 2025. Existing API keys and endpoints continue to work. The Python SDK (`massive` package) defaults to `api.massive.com`; `api.polygon.io` remains supported.

- **Base URL**: `https://api.massive.com` (legacy: `https://api.polygon.io`)
- **Python package**: `massive` (v1.16.3+), requires Python >= 3.9
- **Install**: `uv add massive`

## Authentication

Two methods:

```
# Header (preferred)
Authorization: Bearer <YOUR_API_KEY>

# Query parameter
GET /v2/snapshot/...?apiKey=<YOUR_API_KEY>
```

The Python client handles auth automatically:

```python
from massive import RESTClient

# Reads MASSIVE_API_KEY from environment
client = RESTClient()

# Or pass explicitly
client = RESTClient(api_key="your_key_here")
```

## Rate Limits

| Tier | Limit |
|------|-------|
| **Free** | 5 requests/minute |
| **Paid** | Unlimited (recommended: stay under 100 req/s) |

For FinAlly, we use one snapshot call per poll cycle. At 15-second intervals on the free tier, that's 4 calls/min — safely within limits.

---

## Endpoints Used by FinAlly

### Primary: Full Market Snapshot

Gets current prices for **multiple tickers in a single API call**. This is the only endpoint FinAlly polls.

```
GET /v2/snapshot/locale/us/markets/stocks/tickers
```

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tickers` | string (comma-separated) | No | Specific tickers. Empty = all tickers. Case-sensitive. |
| `include_otc` | boolean | No | Include OTC securities (default: false) |

**Response:**

```json
{
  "count": 2,
  "status": "OK",
  "tickers": [
    {
      "ticker": "AAPL",
      "todaysChange": -1.23,
      "todaysChangePerc": -0.65,
      "updated": 1675190399000000000,
      "day": {
        "o": 129.61,
        "h": 130.15,
        "l": 125.07,
        "c": 125.07,
        "v": 111237700,
        "vw": 127.35
      },
      "prevDay": {
        "o": 128.50,
        "h": 130.00,
        "l": 127.80,
        "c": 129.61,
        "v": 98000000,
        "vw": 129.10
      },
      "min": {
        "o": 125.10,
        "h": 125.15,
        "l": 125.05,
        "c": 125.07,
        "v": 50000,
        "vw": 125.09,
        "av": 111237700,
        "n": 120,
        "t": 1675190340000
      },
      "lastTrade": {
        "p": 125.07,
        "s": 100,
        "x": 4,
        "c": [37],
        "i": "118749",
        "t": 1675190399000000000
      },
      "lastQuote": {
        "P": 125.08,
        "S": 1000,
        "p": 125.06,
        "s": 500,
        "t": 1675190399500000000
      }
    }
  ]
}
```

**Key fields we use:**

| Field | Description |
|-------|-------------|
| `ticker` | Ticker symbol |
| `lastTrade.p` | Last trade price (the "current price") |
| `lastTrade.t` | Nanosecond timestamp of last trade |
| `todaysChange` | Pre-computed price change (today vs previous close) |
| `todaysChangePerc` | Pre-computed percentage change |
| `day.o/h/l/c` | Today's OHLC |
| `prevDay.c` | Previous day's close |
| `lastQuote.p` | Bid price |
| `lastQuote.P` | Ask price |

**Python client usage:**

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key="your_key")

snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],
)

for snap in snapshots:
    print(f"{snap.ticker}: ${snap.last_trade.price}")
    print(f"  Day change: {snap.todays_change_perc}%")
```

Note: `get_snapshot_all()` is synchronous. In async code, run it via `asyncio.to_thread()`.

---

## Other Available Endpoints (Reference)

These are not used by FinAlly but documented for completeness.

### Single Ticker Snapshot

```
GET /v2/snapshot/locale/us/markets/stocks/tickers/{stocksTicker}
```

Same structure as a single element from the full snapshot's `tickers` array.

```python
snapshot = client.get_snapshot_ticker(
    market_type=SnapshotMarketType.STOCKS,
    ticker="AAPL",
)
print(f"Price: ${snapshot.last_trade.price}")
```

### Last Trade

```
GET /v2/last/trade/{stocksTicker}
```

Response: `results.p` = price, `results.s` = size, `results.t` = nanosecond timestamp.

```python
trade = client.get_last_trade(ticker="AAPL")
print(f"Last trade: ${trade.price} x {trade.size}")
```

### Last Quote (NBBO)

```
GET /v2/last/nbbo/{stocksTicker}
```

Response: `results.p` = bid, `results.P` = ask, `results.s` = bid size, `results.S` = ask size.

```python
quote = client.get_last_quote(ticker="AAPL")
print(f"Bid: ${quote.bid} Ask: ${quote.ask}")
```

### Aggregates (Historical OHLCV Bars)

```
GET /v2/aggs/ticker/{stocksTicker}/range/{multiplier}/{timespan}/{from}/{to}
```

- `timespan`: `minute`, `hour`, `day`, `week`, `month`, `quarter`, `year`
- `from`/`to`: YYYY-MM-DD or millisecond timestamp
- Response `results[]`: `o`=open, `h`=high, `l`=low, `c`=close, `v`=volume, `vw`=VWAP, `t`=Unix ms

```python
aggs = list(client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2024-01-01",
    to="2024-01-31",
))
for a in aggs:
    print(f"{a.timestamp}: O={a.open} H={a.high} L={a.low} C={a.close}")
```

### Previous Day Bar

```
GET /v2/aggs/ticker/{stocksTicker}/prev
```

Same structure as aggregates, single result.

### Daily Market Summary (Grouped)

```
GET /v2/aggs/grouped/locale/us/market/stocks/{date}
```

Returns all tickers' daily OHLCV for a given date. Each result includes a `T` field for ticker symbol.

---

## Error Handling

| HTTP Code | Meaning | Action |
|-----------|---------|--------|
| 200 | Success | Process response |
| 401 | Invalid API key | Check `MASSIVE_API_KEY` |
| 403 | Insufficient subscription | Endpoint requires paid tier |
| 429 | Rate limit exceeded | Back off, increase poll interval |
| 5xx | Server error | Retry on next poll cycle |

The Python client raises exceptions for non-200 responses. Our poller catches all exceptions and logs them, continuing to retry on the next interval.

## Timestamp Formats

- **REST response timestamps**: Nanoseconds (divide by 1,000,000,000 for Unix seconds) or milliseconds (divide by 1,000) depending on the field
- `lastTrade.t` and `lastQuote.t`: nanoseconds
- `min.t` and aggregate `t` fields: milliseconds
- Our code converts to Unix seconds before storing in the `PriceCache`
