# Market Data Interface

This document describes the unified Python API for retrieving stock prices in FinAlly. All downstream code (SSE streaming, portfolio valuation, trade execution) is source-agnostic — it reads from `PriceCache` and never calls a data source directly.

**Implementation location**: `backend/app/market/`

---

## Architecture

```
┌──────────────────────────────────────────────────────┐
│  create_market_data_source(cache)   [factory.py]     │
│       │                                              │
│       ├── MASSIVE_API_KEY set? ──► MassiveDataSource │
│       └── otherwise           ──► SimulatorDataSource│
└──────────────────────────────────────────────────────┘
                 │
                 │ writes to
                 ▼
          ┌────────────┐
          │ PriceCache │  (thread-safe, in-memory)
          └────────────┘
                 │
                 │ read by
                 ▼
    ┌────────────────────────┐
    │  SSE /api/stream/prices│
    │  Portfolio valuation   │
    │  Trade execution       │
    └────────────────────────┘
```

---

## Data Model: `PriceUpdate`

`PriceUpdate` is the single shared data type that flows from any data source to all consumers.

```python
# backend/app/market/models.py
from dataclasses import dataclass, field
import time

@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float = field(default_factory=time.time)  # Unix seconds

    @property
    def change(self) -> float:
        """Absolute price change from previous update."""
        return round(self.price - self.previous_price, 4)

    @property
    def change_percent(self) -> float:
        """Percentage change from previous update."""
        if self.previous_price == 0:
            return 0.0
        return round((self.price - self.previous_price) / self.previous_price * 100, 4)

    @property
    def direction(self) -> str:
        """'up', 'down', or 'flat'."""
        if self.price > self.previous_price:
            return "up"
        elif self.price < self.previous_price:
            return "down"
        return "flat"

    def to_dict(self) -> dict:
        """Serialize for JSON / SSE transmission."""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "previous_price": self.previous_price,
            "timestamp": self.timestamp,
            "change": self.change,
            "change_percent": self.change_percent,
            "direction": self.direction,
        }
```

**Notes:**
- `previous_price` equals `price` on the very first update for a ticker, so `direction` is `"flat"` and the frontend does not flash.
- `timestamp` is Unix epoch seconds (not milliseconds). The Massive client converts from milliseconds.
- The dataclass is frozen (immutable) — consumers can safely hold references without copies.

---

## The Abstract Interface: `MarketDataSource`

```python
# backend/app/market/interface.py
from abc import ABC, abstractmethod

class MarketDataSource(ABC):
    """Contract for market data providers.

    Implementations push price updates into a shared PriceCache.
    Downstream code reads from the cache; it never calls the source directly.

    Lifecycle:
        source = create_market_data_source(cache)
        await source.start(["AAPL", "GOOGL", ...])
        await source.add_ticker("TSLA")
        await source.remove_ticker("GOOGL")
        await source.stop()
    """

    @abstractmethod
    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates. Starts an internal background task."""

    @abstractmethod
    async def stop(self) -> None:
        """Stop the background task. Safe to call multiple times."""

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """Add a ticker. The next update cycle includes it."""

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """Remove a ticker. Also removes it from PriceCache."""

    @abstractmethod
    def get_tickers(self) -> list[str]:
        """Return the current list of tracked tickers."""
```

All five methods must be implemented. `start()` is called once at app startup; `stop()` once at shutdown. `add_ticker` / `remove_ticker` are called when the user modifies the watchlist.

---

## The Cache: `PriceCache`

```python
# backend/app/market/cache.py
from threading import Lock
from .models import PriceUpdate
import time

class PriceCache:
    """Thread-safe in-memory cache of the latest price per ticker.

    Writers: one MarketDataSource (either simulator or Massive poller).
    Readers: SSE endpoint (async), portfolio routes (sync-in-async), trade execution.
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = Lock()
        self._version: int = 0  # Incremented on every write

    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate:
        """Write a new price. Computes direction from the previous value.

        First update for a ticker: previous_price == price, direction == 'flat'.
        """
        with self._lock:
            ts = timestamp or time.time()
            prev = self._prices.get(ticker)
            previous_price = prev.price if prev else price
            update = PriceUpdate(ticker=ticker, price=round(price, 2),
                                 previous_price=round(previous_price, 2), timestamp=ts)
            self._prices[ticker] = update
            self._version += 1
            return update

    def get(self, ticker: str) -> PriceUpdate | None:
        with self._lock:
            return self._prices.get(ticker)

    def get_price(self, ticker: str) -> float | None:
        update = self.get(ticker)
        return update.price if update else None

    def get_all(self) -> dict[str, PriceUpdate]:
        """Snapshot of all prices. Returns a shallow copy."""
        with self._lock:
            return dict(self._prices)

    def remove(self, ticker: str) -> None:
        with self._lock:
            self._prices.pop(ticker, None)

    @property
    def version(self) -> int:
        """Monotonic counter. Increments on every update. Used by SSE for change detection."""
        return self._version
```

**SSE change detection**: The SSE generator compares `cache.version` against its last-seen version. It only sends a new event when the version has changed, avoiding redundant transmissions.

---

## The Factory: `create_market_data_source`

```python
# backend/app/market/factory.py
import os
from .cache import PriceCache
from .interface import MarketDataSource
from .massive_client import MassiveDataSource
from .simulator import SimulatorDataSource

def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """Select data source based on MASSIVE_API_KEY environment variable.

    - Set and non-empty → MassiveDataSource (real market data, REST polling)
    - Absent or empty   → SimulatorDataSource (GBM simulation, no network)
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        return MassiveDataSource(api_key=api_key, price_cache=price_cache)
    else:
        return SimulatorDataSource(price_cache=price_cache)
```

---

## `MassiveDataSource` Implementation

Wraps the `massive.RESTClient` with asyncio-friendly polling. Because the Massive Python client is synchronous, calls run in a thread via `asyncio.to_thread()`.

```python
# backend/app/market/massive_client.py
import asyncio
import logging
from massive import RESTClient
from massive.rest.models import SnapshotMarketType
from .cache import PriceCache
from .interface import MarketDataSource

class MassiveDataSource(MarketDataSource):
    """Polls GET /v2/snapshot/locale/us/markets/stocks/tickers on an interval."""

    def __init__(self, api_key: str, price_cache: PriceCache,
                 poll_interval: float = 15.0) -> None:
        self._api_key = api_key
        self._cache = price_cache
        self._interval = poll_interval
        self._tickers: list[str] = []
        self._task: asyncio.Task | None = None
        self._client: RESTClient | None = None

    async def start(self, tickers: list[str]) -> None:
        self._client = RESTClient(api_key=self._api_key)
        self._tickers = list(tickers)
        await self._poll_once()       # Populate cache immediately
        self._task = asyncio.create_task(self._poll_loop(), name="massive-poller")

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass

    async def add_ticker(self, ticker: str) -> None:
        if ticker.upper() not in self._tickers:
            self._tickers.append(ticker.upper())

    async def remove_ticker(self, ticker: str) -> None:
        self._tickers = [t for t in self._tickers if t != ticker.upper()]
        self._cache.remove(ticker.upper())

    def get_tickers(self) -> list[str]:
        return list(self._tickers)

    async def _poll_loop(self) -> None:
        while True:
            await asyncio.sleep(self._interval)
            await self._poll_once()

    async def _poll_once(self) -> None:
        if not self._tickers or not self._client:
            return
        try:
            snapshots = await asyncio.to_thread(self._fetch_snapshots)
            for snap in snapshots:
                try:
                    price = snap.last_trade.price
                    ts = snap.last_trade.timestamp / 1000.0  # ms → seconds
                    self._cache.update(ticker=snap.ticker, price=price, timestamp=ts)
                except (AttributeError, TypeError):
                    pass  # Skip tickers with no trade data
        except Exception as e:
            logging.getLogger(__name__).error("Massive poll failed: %s", e)

    def _fetch_snapshots(self) -> list:
        return self._client.get_snapshot_all(
            market_type=SnapshotMarketType.STOCKS,
            tickers=self._tickers,
        )
```

**Key design decisions:**

- `asyncio.to_thread()` keeps the event loop unblocked during HTTP calls.
- First poll runs synchronously in `start()` so the cache is populated before the first SSE client connects.
- Errors are caught and logged without re-raising — the loop will retry on the next interval. A 429 or transient network error does not crash the server.
- `poll_interval=15.0` (seconds) is the safe default for the Massive free tier (5 req/min limit). Pass a lower value when using a paid key.

---

## `SimulatorDataSource` Implementation

See `MARKET_SIMULATOR.md` for the full simulator design. The `SimulatorDataSource` wraps `GBMSimulator` and satisfies the same `MarketDataSource` interface.

```python
class SimulatorDataSource(MarketDataSource):
    """Drives GBMSimulator every 500ms, writing results to PriceCache."""

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers)
        # Seed cache immediately with initial prices
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop())

    async def _run_loop(self) -> None:
        while True:
            prices = self._sim.step()          # Dict[ticker, price]
            for ticker, price in prices.items():
                self._cache.update(ticker=ticker, price=price)
            await asyncio.sleep(self._interval)
```

---

## How to Use (Downstream Code)

```python
from app.market import PriceCache, create_market_data_source

# --- App startup (FastAPI lifespan handler) ---
cache = PriceCache()
source = create_market_data_source(cache)   # Reads MASSIVE_API_KEY from env
await source.start(["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA",
                    "NVDA", "META", "JPM", "V", "NFLX"])

# --- Reading prices ---
update = cache.get("AAPL")            # PriceUpdate | None
price  = cache.get_price("AAPL")      # float | None
all_prices = cache.get_all()          # dict[str, PriceUpdate]

# --- Watchlist changes ---
await source.add_ticker("PYPL")
await source.remove_ticker("NFLX")

# --- App shutdown ---
await source.stop()
```

---

## SSE Streaming

The SSE router is created from the cache and mounted into the FastAPI app:

```python
from app.market import create_stream_router

router = create_stream_router(price_cache)
app.include_router(router)
# Registers: GET /api/stream/prices
```

The generator sends all cached prices as one JSON object whenever `cache.version` changes:

```
data: {"AAPL": {"ticker":"AAPL","price":190.50,"previous_price":190.20,"timestamp":1713801600.123,"change":0.3,"change_percent":0.1574,"direction":"up"}, ...}
```

The client uses the browser's native `EventSource` API and reconnects automatically (no custom retry logic needed).

---

## Public Module Exports

```python
# backend/app/market/__init__.py
from .cache import PriceCache
from .factory import create_market_data_source
from .interface import MarketDataSource
from .models import PriceUpdate
from .stream import create_stream_router

__all__ = [
    "PriceCache",
    "create_market_data_source",
    "MarketDataSource",
    "PriceUpdate",
    "create_stream_router",
]
```

All downstream code imports from `app.market` — never from the submodules directly.

---

## Configuration Reference

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `MASSIVE_API_KEY` | (empty) | If set, uses real market data. Absent → simulator. |

`MassiveDataSource` constructor also accepts `poll_interval` (float, seconds) for tuning. This is not currently exposed as an environment variable but could be if tiered polling is needed.
