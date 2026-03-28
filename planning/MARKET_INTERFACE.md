# Market Data Interface — Unified Python API

Design of the unified interface for retrieving stock prices, abstracting over the Massive API and the built-in simulator.

## Design Pattern: Strategy

Both data sources implement the same abstract interface. A factory function selects the implementation based on an environment variable. All downstream code (SSE streaming, portfolio valuation, trade execution) reads from a shared `PriceCache` and never touches the data source directly.

```
                    ┌─────────────────────┐
                    │  MarketDataSource    │  (ABC)
                    │  interface.py        │
                    └──────┬──────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
   ┌──────────┴──────────┐  ┌──────────┴──────────┐
   │ SimulatorDataSource  │  │ MassiveDataSource    │
   │ simulator.py         │  │ massive_client.py    │
   └──────────┬──────────┘  └──────────┬──────────┘
              │                         │
              └────────────┬────────────┘
                           │  writes to
                    ┌──────▼──────────────┐
                    │  PriceCache          │  (thread-safe)
                    │  cache.py            │
                    └──────┬──────────────┘
                           │  reads from
              ┌────────────┼────────────┐
              │            │            │
         SSE stream   Portfolio    Trade exec
```

## Modules

All modules live in `backend/app/market/`.

| Module | Class/Function | Purpose |
|--------|---------------|---------|
| `models.py` | `PriceUpdate` | Immutable dataclass for a single price point |
| `interface.py` | `MarketDataSource` | Abstract base class |
| `cache.py` | `PriceCache` | Thread-safe in-memory price store |
| `simulator.py` | `GBMSimulator`, `SimulatorDataSource` | GBM price engine + async wrapper |
| `massive_client.py` | `MassiveDataSource` | REST polling client |
| `seed_prices.py` | Constants | Seed prices, GBM params, correlation groups |
| `factory.py` | `create_market_data_source()` | Environment-based factory |
| `stream.py` | `create_stream_router()` | FastAPI SSE endpoint factory |

## Abstract Interface

```python
# interface.py

class MarketDataSource(ABC):

    async def start(self, tickers: list[str]) -> None:
        """Begin producing price updates. Starts background task.
        Must be called exactly once."""

    async def stop(self) -> None:
        """Stop background task, release resources. Safe to call multiple times."""

    async def add_ticker(self, ticker: str) -> None:
        """Add ticker to active set. No-op if already present."""

    async def remove_ticker(self, ticker: str) -> None:
        """Remove ticker from active set and price cache. No-op if not present."""

    def get_tickers(self) -> list[str]:
        """Return currently tracked tickers."""
```

**Key contract**: Implementations push `PriceUpdate` objects into the `PriceCache` on their own schedule. Consumers never call the data source for prices — they read from the cache.

## PriceUpdate Model

```python
# models.py

@dataclass(frozen=True, slots=True)
class PriceUpdate:
    ticker: str
    price: float
    previous_price: float
    timestamp: float          # Unix seconds

    # Computed properties
    change -> float           # price - previous_price
    change_percent -> float   # percentage change
    direction -> str          # "up", "down", or "flat"
    to_dict() -> dict         # JSON-serializable dict
```

## PriceCache

```python
# cache.py

class PriceCache:
    def update(self, ticker: str, price: float, timestamp: float | None = None) -> PriceUpdate
    def get(self, ticker: str) -> PriceUpdate | None
    def get_price(self, ticker: str) -> float | None
    def get_all(self) -> dict[str, PriceUpdate]
    def remove(self, ticker: str) -> None

    @property
    def version(self) -> int   # Monotonic counter, bumped on every update
```

The `version` property enables efficient SSE change detection — the stream endpoint only sends data when the version has changed since the last push.

## Factory

```python
# factory.py

def create_market_data_source(price_cache: PriceCache) -> MarketDataSource:
    """MASSIVE_API_KEY set and non-empty -> MassiveDataSource
       Otherwise -> SimulatorDataSource"""
```

No configuration beyond the environment variable. Returns an unstarted source.

## Usage Pattern

```python
from app.market import PriceCache, create_market_data_source

# Application startup
cache = PriceCache()
source = create_market_data_source(cache)
await source.start(["AAPL", "GOOGL", "MSFT", "AMZN", "TSLA",
                     "NVDA", "META", "JPM", "V", "NFLX"])

# Read current prices (from any async handler)
update = cache.get("AAPL")          # PriceUpdate or None
price = cache.get_price("AAPL")     # float or None
all_prices = cache.get_all()        # dict[str, PriceUpdate]

# Dynamic watchlist changes
await source.add_ticker("PYPL")
await source.remove_ticker("NFLX")

# Application shutdown
await source.stop()
```

## Implementation Details by Source

### SimulatorDataSource

- Creates a `GBMSimulator` instance on `start()`
- Runs an `asyncio.Task` that calls `simulator.step()` every 500ms
- Each `step()` advances all tickers by one GBM time step with correlated random draws
- Seeds the cache with initial prices immediately so SSE has data on first connect
- `add_ticker()` / `remove_ticker()` delegate to the simulator and rebuild the Cholesky correlation matrix

### MassiveDataSource

- Creates a `massive.RESTClient` on `start()`
- Runs an `asyncio.Task` that polls every `poll_interval` seconds (default 15s)
- Calls `client.get_snapshot_all()` with all tracked tickers (one API call)
- Runs the synchronous REST call via `asyncio.to_thread()` to avoid blocking
- Extracts `last_trade.price` and timestamp from each snapshot result
- Writes to the cache; logs and continues on errors (bad key, rate limit, network)

### Both Implementations

- Background task is an `asyncio.Task` created via `asyncio.create_task()`
- `stop()` cancels the task and awaits `CancelledError`
- `add_ticker()` / `remove_ticker()` are safe to call while the source is running
- `remove_ticker()` also removes the ticker from the `PriceCache`

## SSE Streaming

The `create_stream_router(cache)` factory returns a FastAPI router with:

```
GET /api/stream/prices   (text/event-stream)
```

The endpoint runs a long-lived async generator that:
1. Checks `cache.version` against the last-seen version
2. If changed, sends a JSON event with all current prices
3. Sleeps briefly, then loops

The client connects with the browser's native `EventSource` API, which handles automatic reconnection.

## Design Decisions

| Decision | Rationale |
|----------|-----------|
| Strategy pattern (ABC) | Downstream code is source-agnostic; easy to test with either implementation |
| PriceCache as mediator | Decouples producers from consumers; single point of truth |
| Thread-safe cache with Lock | The Massive client runs in a thread via `to_thread()`; cache must be safe |
| Version counter on cache | Avoids sending duplicate SSE events when no prices have changed |
| Factory reads env var | Zero configuration — set the key for real data, omit for simulation |
| Frozen dataclass for PriceUpdate | Immutability prevents accidental mutation across async boundaries |
