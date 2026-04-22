# Massive API Reference

Massive (formerly Polygon.io, rebranded October 2025) is the market data provider used when `MASSIVE_API_KEY` is set. This document covers the REST endpoints relevant to this project and how to use the official Python client.

**API base**: `https://api.massive.com` (legacy `https://api.polygon.io` still works)  
**Python package**: `massive` (the official Python client; was `polygon-api-client`)  
**Docs**: https://massive.com/docs/rest/stocks/overview

---

## Authentication

All requests require an API key passed as the `apiKey` query parameter or via the `Authorization: Bearer <key>` header.

```python
from massive import RESTClient

client = RESTClient(api_key="YOUR_MASSIVE_API_KEY")
```

Set the key via environment variable to keep it out of code:

```python
import os
from massive import RESTClient

client = RESTClient(api_key=os.environ["MASSIVE_API_KEY"])
```

---

## Rate Limits

| Tier | Requests / min | Typical use case |
|------|---------------|-----------------|
| Free | 5 | Development; poll every 15s |
| Starter ($29/mo) | unlimited | Poll every 5–10s |
| Developer ($79/mo) | unlimited | Poll every 2–5s |
| Advanced | unlimited | Near-realtime polling |

Free tier returns **15-minute delayed** data for most stocks endpoints. Paid tiers receive real-time data.

**This project default**: poll every `15.0` seconds on the free tier (configurable via `poll_interval`).

---

## Key Endpoints

### 1. Full Market Snapshot (Multi-Ticker) — Primary Endpoint

Retrieve last trade, last quote, daily bar, and previous day bar for a list of tickers in a single request. This is the most efficient approach for polling multiple tickers.

```
GET /v2/snapshot/locale/us/markets/stocks/tickers
```

**Query parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `tickers` | string (comma-separated) | No | Filter to specific tickers: `AAPL,TSLA,MSFT`. Omit to fetch all. |
| `include_otc` | boolean | No | Include OTC securities (default: false) |
| `apiKey` | string | Yes | Your API key |

**Response structure:**

```json
{
  "status": "OK",
  "count": 2,
  "tickers": [
    {
      "ticker": "AAPL",
      "todaysChange": 2.50,
      "todaysChangePerc": 1.34,
      "updated": 1713801600000,
      "day": {
        "o": 187.00,
        "h": 192.50,
        "l": 186.20,
        "c": 190.50,
        "v": 54321000,
        "vw": 189.42
      },
      "prevDay": {
        "o": 185.00,
        "h": 188.70,
        "l": 184.10,
        "c": 188.00,
        "v": 48900000,
        "vw": 186.55
      },
      "min": {
        "o": 190.20,
        "h": 190.80,
        "l": 190.10,
        "c": 190.50,
        "v": 123456
      },
      "lastTrade": {
        "p": 190.50,
        "s": 100,
        "t": 1713801600000,
        "x": 4
      },
      "lastQuote": {
        "P": 190.55,
        "S": 1,
        "p": 190.45,
        "s": 2,
        "t": 1713801600000
      }
    }
  ]
}
```

**Field reference:**

| Field | Description |
|-------|-------------|
| `todaysChangePerc` | % change from prior day close |
| `updated` | Last update timestamp (Unix milliseconds) |
| `day.c` | Current day's close (latest bar close) |
| `prevDay.c` | Previous trading day's close |
| `lastTrade.p` | Last trade price |
| `lastTrade.t` | Last trade timestamp (Unix milliseconds) |
| `lastQuote.P` | Ask price |
| `lastQuote.p` | Bid price |

**Python example using the official client:**

```python
from massive import RESTClient
from massive.rest.models import SnapshotMarketType

client = RESTClient(api_key="YOUR_KEY")

# Fetch snapshots for specific tickers
snapshots = client.get_snapshot_all(
    market_type=SnapshotMarketType.STOCKS,
    tickers=["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA"],
)

for snap in snapshots:
    price = snap.last_trade.price
    ts = snap.last_trade.timestamp / 1000.0  # ms → seconds
    change_pct = snap.todays_change_perc
    print(f"{snap.ticker}: ${price:.2f}  ({change_pct:+.2f}%)")
```

**Raw HTTP example (no client library):**

```python
import requests

resp = requests.get(
    "https://api.massive.com/v2/snapshot/locale/us/markets/stocks/tickers",
    params={
        "tickers": "AAPL,GOOGL,MSFT",
        "apiKey": "YOUR_KEY",
    },
)
data = resp.json()

for ticker_data in data["tickers"]:
    ticker = ticker_data["ticker"]
    price = ticker_data["lastTrade"]["p"]
    ts_ms = ticker_data["lastTrade"]["t"]
    print(f"{ticker}: ${price:.2f}")
```

---

### 2. Single Ticker Snapshot

For a single ticker. Less efficient than the multi-ticker batch call above when watching many tickers.

```
GET /v2/snapshot/locale/us/markets/stocks/tickers/{stocksTicker}
```

```python
snap = client.get_snapshot(market_type=SnapshotMarketType.STOCKS, ticker="AAPL")
print(snap.last_trade.price)
```

---

### 3. Last Trade

Retrieves the latest single trade for one ticker. Useful for verifying a price independently of the snapshot.

```
GET /v2/last/trade/{stocksTicker}
```

```python
trade = client.get_last_trade("AAPL")
print(f"Last trade: ${trade.price} at {trade.timestamp}")
```

---

### 4. Previous Day Close

Retrieves the OHLCV bar for the prior trading day. Use this to calculate "daily change %" on startup.

```
GET /v2/aggs/ticker/{stocksTicker}/prev
```

```python
prev = client.get_previous_close("AAPL", adjusted=True)
for bar in prev:
    print(f"Prev close: ${bar.close}  Open: ${bar.open}")
```

---

### 5. Historical Aggregates (OHLCV Bars)

Useful for building charts. Returns bars at any interval (1min, 1day, etc.).

```
GET /v2/aggs/ticker/{stocksTicker}/range/{multiplier}/{timespan}/{from}/{to}
```

```python
from datetime import date

bars = client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="minute",
    from_=date(2026, 4, 1),
    to=date(2026, 4, 22),
    adjusted=True,
    sort="asc",
    limit=500,
)

for bar in bars:
    print(f"{bar.timestamp}: open={bar.open} close={bar.close} vol={bar.volume}")
```

---

## Error Handling

The Massive client raises exceptions on HTTP errors. Common error codes:

| HTTP Code | Meaning | Action |
|-----------|---------|--------|
| 401 | Invalid API key | Check `MASSIVE_API_KEY` |
| 403 | Endpoint not on your plan | Upgrade or use a different endpoint |
| 429 | Rate limit exceeded | Back off; increase poll interval |
| 403 | Delayed data — key lacks real-time | Upgrade plan |

```python
try:
    snapshots = client.get_snapshot_all(
        market_type=SnapshotMarketType.STOCKS,
        tickers=tickers,
    )
except Exception as e:
    logger.error("Massive poll failed: %s", e)
    # Continue — retry on next interval
```

---

## Choosing the Right Endpoint

| Goal | Endpoint | Notes |
|------|----------|-------|
| Poll multiple tickers at once | `GET /v2/snapshot/locale/us/markets/stocks/tickers` | **Use this** — single call for all tickers |
| Daily open price (for % change calc) | `GET /v2/aggs/ticker/{t}/prev` | Fetch once at startup |
| Historical bars for main chart | `GET /v2/aggs/ticker/{t}/range/...` | On demand per ticker |
| Single ticker quick check | `GET /v2/last/trade/{t}` | Less efficient than batch snapshot |

For FinAlly, the **Full Market Snapshot** endpoint is the workhorse: one API call returns last trade prices, previous day data, and daily change for all watched tickers simultaneously.

---

## Installing the Client

```toml
# backend/pyproject.toml
[project]
dependencies = [
    "massive",   # Official Massive/Polygon.io Python client
]
```

```bash
cd backend && uv sync
```

The `massive` package is a renamed continuation of `polygon-api-client`. Import paths remain under `massive.*`.
