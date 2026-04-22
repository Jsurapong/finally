# Market Simulator

The market simulator generates realistic-looking stock prices without any external API calls. It is the default data source when `MASSIVE_API_KEY` is not set.

**Implementation location**: `backend/app/market/simulator.py` and `backend/app/market/seed_prices.py`

---

## Design Goals

- Realistic price movement — not just random noise
- Correlated moves across related tickers (tech stocks move together)
- Occasional dramatic events for visual drama in the UI
- Configurable volatility and drift per ticker
- Sub-millisecond per-tick computation (hot path, called every 500ms)
- No external dependencies (pure Python + NumPy)

---

## Mathematical Model: Geometric Brownian Motion

The simulator uses **Geometric Brownian Motion (GBM)**, the standard model for stock prices in quantitative finance. GBM has two key properties that make it suitable here:

1. Prices are always positive (no negative prices)
2. Returns are log-normally distributed (realistic for equities)

**Discrete-time GBM formula:**

```
S(t + dt) = S(t) * exp((μ - σ²/2) * dt  +  σ * √dt * Z)
```

Where:

| Symbol | Meaning |
|--------|---------|
| `S(t)` | Current price |
| `μ` (mu) | Annualized drift / expected return (e.g., 0.05 = 5% per year) |
| `σ` (sigma) | Annualized volatility (e.g., 0.25 = 25% annual std dev) |
| `dt` | Time step as a fraction of a trading year |
| `Z` | Standard normal random variable (correlated across tickers) |

**Time step size:**

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # = 5,896,800 seconds

# For 500ms ticks:
dt = 0.5 / TRADING_SECONDS_PER_YEAR  # ≈ 8.48e-8
```

This tiny `dt` means each tick produces sub-cent price moves that accumulate naturally and realistically over time. Over a simulated "trading day" (~23,400 ticks at 500ms) the total volatility matches the configured annual sigma.

---

## Correlated Price Moves

Real stocks in the same sector move together — when AAPL drops, MSFT usually drops too. The simulator reproduces this using **Cholesky decomposition** of a sector-based correlation matrix.

### How It Works

**Step 1**: Build a correlation matrix based on sector membership.

```
Sector   | Intra-sector correlation
---------|------------------------
Tech     | 0.6  (AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX)
Finance  | 0.5  (JPM, V)
TSLA     | 0.3  (with everything — TSLA does its own thing)
Cross    | 0.3  (between sectors, or unknown tickers)
```

Example correlation matrix for 3 tickers (AAPL, MSFT, JPM):

```
       AAPL  MSFT  JPM
AAPL [  1.0   0.6   0.3 ]
MSFT [  0.6   1.0   0.3 ]
JPM  [  0.3   0.3   1.0 ]
```

**Step 2**: Compute the Cholesky decomposition `L` such that `L @ L.T == corr_matrix`.

**Step 3**: At each tick, generate `n` independent standard normal draws `Z_ind`, then apply:

```
Z_corr = L @ Z_ind
```

`Z_corr` is now a vector of correlated normal draws. Each ticker's price step uses its corresponding `Z_corr[i]`.

**Step 4**: Apply the GBM formula using `Z_corr[i]` for ticker `i`.

### Implementation

```python
import numpy as np
import math

class GBMSimulator:
    DEFAULT_DT = 0.5 / (252 * 6.5 * 3600)  # 500ms tick

    def __init__(self, tickers: list[str], dt: float = DEFAULT_DT,
                 event_probability: float = 0.001) -> None:
        self._dt = dt
        self._event_prob = event_probability
        self._tickers: list[str] = []
        self._prices: dict[str, float] = {}
        self._params: dict[str, dict[str, float]] = {}
        self._cholesky: np.ndarray | None = None

        for ticker in tickers:
            self._add_ticker_internal(ticker)
        self._rebuild_cholesky()

    def step(self) -> dict[str, float]:
        """Advance all tickers by one dt. Returns {ticker: new_price}."""
        n = len(self._tickers)
        if n == 0:
            return {}

        # Correlated normal draws
        z_ind = np.random.standard_normal(n)
        z_corr = self._cholesky @ z_ind if self._cholesky is not None else z_ind

        result = {}
        for i, ticker in enumerate(self._tickers):
            mu = self._params[ticker]["mu"]
            sigma = self._params[ticker]["sigma"]

            drift = (mu - 0.5 * sigma**2) * self._dt
            diffusion = sigma * math.sqrt(self._dt) * z_corr[i]
            self._prices[ticker] *= math.exp(drift + diffusion)

            # Random event shock
            if random.random() < self._event_prob:
                shock = random.uniform(0.02, 0.05) * random.choice([-1, 1])
                self._prices[ticker] *= (1 + shock)

            result[ticker] = round(self._prices[ticker], 2)

        return result
```

The Cholesky is rebuilt (`O(n²)`) only when tickers are added or removed — not on every tick. With `n < 50`, this is negligible.

---

## Seed Prices and Per-Ticker Parameters

Each ticker has a realistic starting price and calibrated GBM parameters:

```python
# backend/app/market/seed_prices.py

SEED_PRICES = {
    "AAPL": 190.00,
    "GOOGL": 175.00,
    "MSFT": 420.00,
    "AMZN": 185.00,
    "TSLA": 250.00,
    "NVDA": 800.00,
    "META": 500.00,
    "JPM":  195.00,
    "V":    280.00,
    "NFLX": 600.00,
}

# sigma: annualized volatility | mu: annualized drift
TICKER_PARAMS = {
    "AAPL": {"sigma": 0.22, "mu": 0.05},   # Stable large cap
    "GOOGL": {"sigma": 0.25, "mu": 0.05},
    "MSFT": {"sigma": 0.20, "mu": 0.05},   # Lowest volatility tech
    "AMZN": {"sigma": 0.28, "mu": 0.05},
    "TSLA": {"sigma": 0.50, "mu": 0.03},   # High vol, lower drift
    "NVDA": {"sigma": 0.40, "mu": 0.08},   # High vol, strong drift
    "META": {"sigma": 0.30, "mu": 0.05},
    "JPM":  {"sigma": 0.18, "mu": 0.04},   # Low vol bank stock
    "V":    {"sigma": 0.17, "mu": 0.04},   # Lowest vol overall
    "NFLX": {"sigma": 0.35, "mu": 0.05},
}

# Fallback for dynamically added tickers not in the list above
DEFAULT_PARAMS = {"sigma": 0.25, "mu": 0.05}

CORRELATION_GROUPS = {
    "tech":    {"AAPL", "GOOGL", "MSFT", "AMZN", "META", "NVDA", "NFLX"},
    "finance": {"JPM", "V"},
}

INTRA_TECH_CORR  = 0.6
INTRA_FINANCE_CORR = 0.5
CROSS_GROUP_CORR = 0.3
TSLA_CORR        = 0.3   # TSLA treated independently
```

**Parameter tuning guide:**

| sigma value | Typical behavior | Examples |
|-------------|-----------------|---------|
| 0.15–0.20 | Slow, stable moves | Utilities, blue-chip banks |
| 0.20–0.30 | Normal large-cap tech | AAPL, MSFT |
| 0.30–0.40 | Elevated volatility | META, NFLX |
| 0.40–0.60 | High-volatility names | NVDA, TSLA |

**Annualized drift `mu`** reflects a modest expected return. Values 0.03–0.08 (3–8% per year) are realistic. Over a short demo session these have negligible effect; they matter for multi-day simulations.

---

## Random Shock Events

A small probability (`0.001` by default = 0.1% chance per tick per ticker) triggers a sudden 2–5% move in either direction:

```python
if random.random() < self._event_prob:
    shock_magnitude = random.uniform(0.02, 0.05)
    shock_sign = random.choice([-1, 1])
    self._prices[ticker] *= 1 + shock_magnitude * shock_sign
```

With 10 tickers at 2 ticks/second, the expected frequency is roughly one event every ~50 seconds — visible but not overwhelming. This creates the dramatic price flashes that make the UI feel alive.

---

## Dynamic Ticker Management

Tickers can be added and removed at runtime via the `MarketDataSource` interface. The Cholesky decomposition is rebuilt each time:

```python
def add_ticker(self, ticker: str) -> None:
    if ticker in self._prices:
        return
    self._tickers.append(ticker)
    self._prices[ticker] = SEED_PRICES.get(ticker, random.uniform(50.0, 300.0))
    self._params[ticker] = TICKER_PARAMS.get(ticker, dict(DEFAULT_PARAMS))
    self._rebuild_cholesky()

def remove_ticker(self, ticker: str) -> None:
    if ticker not in self._prices:
        return
    self._tickers.remove(ticker)
    del self._prices[ticker]
    del self._params[ticker]
    self._rebuild_cholesky()
```

Unknown tickers get a random seed price between $50–$300 and default parameters. This ensures the system always has a starting price even for tickers not in the predefined list.

---

## `SimulatorDataSource` — The Async Wrapper

`GBMSimulator` is a pure synchronous class. `SimulatorDataSource` wraps it with an asyncio background loop and satisfies the `MarketDataSource` interface:

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache: PriceCache,
                 update_interval: float = 0.5,
                 event_probability: float = 0.001) -> None:
        self._cache = price_cache
        self._interval = update_interval
        self._event_prob = event_probability
        self._sim: GBMSimulator | None = None
        self._task: asyncio.Task | None = None

    async def start(self, tickers: list[str]) -> None:
        self._sim = GBMSimulator(tickers, event_probability=self._event_prob)
        # Seed cache before the loop starts so SSE has data immediately
        for ticker in tickers:
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)
        self._task = asyncio.create_task(self._run_loop(), name="simulator-loop")

    async def _run_loop(self) -> None:
        while True:
            try:
                prices = self._sim.step()
                for ticker, price in prices.items():
                    self._cache.update(ticker=ticker, price=price)
            except Exception:
                logger.exception("Simulator step failed")
            await asyncio.sleep(self._interval)

    async def add_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.add_ticker(ticker)
            price = self._sim.get_price(ticker)
            if price is not None:
                self._cache.update(ticker=ticker, price=price)

    async def remove_ticker(self, ticker: str) -> None:
        if self._sim:
            self._sim.remove_ticker(ticker)
        self._cache.remove(ticker)

    async def stop(self) -> None:
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
```

Key points:
- The cache is seeded immediately in `start()` before the loop begins. The first SSE client will receive prices on their first poll.
- `step()` runs on the asyncio event loop thread. Because it only uses NumPy and Python builtins (no I/O), it completes in microseconds and does not need to be offloaded to a thread.
- Exceptions in `step()` are caught and logged. A bug in the GBM math should not crash the server.

---

## Performance Characteristics

| Metric | Value |
|--------|-------|
| Tick interval | 500ms (2 ticks/second) |
| Per-tick CPU (10 tickers) | < 1ms (dominated by NumPy matrix multiply) |
| Memory per ticker | ~200 bytes (price float + params dict) |
| Cholesky rebuild cost | O(n²) — negligible for n < 50 |
| Event frequency (10 tickers) | ~1 event / 50 seconds |

---

## Testing

The simulator has unit tests in `backend/tests/market/`:

- `test_simulator.py` (17 tests) — GBM math correctness, correlation structure, event generation, add/remove ticker
- `test_simulator_source.py` (10 integration tests) — lifecycle, cache population, async behavior

Run:

```bash
cd backend
uv run --extra dev pytest tests/market/test_simulator.py tests/market/test_simulator_source.py -v
```

---

## Demo

A Rich terminal dashboard runs the simulator standalone:

```bash
cd backend
uv run market_data_demo.py
```

Displays a live-updating table with all 10 tickers, current prices, sparklines, direction arrows, and an event log for shock moves. Runs 60 seconds or until Ctrl+C.
