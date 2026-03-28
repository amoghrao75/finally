# Market Simulator — Design and Code Structure

How the GBM-based stock price simulator works, its mathematical model, and code organization.

## Purpose

The simulator generates realistic-looking stock prices without any external API dependency. It runs by default (when `MASSIVE_API_KEY` is not set) and produces prices that update every 500ms with correlated cross-ticker movements and occasional dramatic events.

## Mathematical Model: Geometric Brownian Motion (GBM)

### Core Formula

```
S(t+dt) = S(t) * exp((mu - 0.5 * sigma^2) * dt + sigma * sqrt(dt) * Z)
```

Where:
- `S(t)` = current price
- `mu` = annualized drift (expected return, e.g. 0.05 = 5%/year)
- `sigma` = annualized volatility (e.g. 0.25 = 25%/year)
- `dt` = time step as a fraction of a trading year
- `Z` = correlated standard normal random variable

### Time Step Calculation

```python
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # = 5,896,800 seconds
dt = 0.5 / TRADING_SECONDS_PER_YEAR          # ~8.48e-8
```

252 trading days, 6.5 hours per day, ticking every 500ms. The tiny `dt` produces sub-cent moves per tick that accumulate naturally over time — realistic micro-movement rather than jumpy steps.

### Why GBM?

- Standard model for stock price dynamics (basis of Black-Scholes)
- Prices can never go negative (log-normal distribution)
- Drift and volatility parameters map directly to intuitive concepts
- Easy to calibrate per ticker (TSLA volatile, V stable)

## Correlated Movements

Real stocks don't move independently — tech stocks tend to move together, finance stocks correlate with each other, and there's a baseline cross-sector correlation.

### Correlation Structure

| Pair | Correlation |
|------|-------------|
| Tech with tech (AAPL, GOOGL, MSFT, AMZN, META, NVDA, NFLX) | 0.6 |
| Finance with finance (JPM, V) | 0.5 |
| TSLA with anything | 0.3 (independent streak) |
| Cross-sector / unknown tickers | 0.3 |

### Cholesky Decomposition

To generate correlated random draws from independent ones:

1. Build an N x N correlation matrix from the pairwise correlations above
2. Compute the Cholesky decomposition: `L = cholesky(corr)` (lower triangular)
3. Generate N independent standard normal draws: `z_independent`
4. Multiply: `z_correlated = L @ z_independent`

The correlated draws are then used in the GBM formula for each ticker. This is rebuilt whenever tickers are added or removed (O(N^2) but N < 50, so negligible).

## Random Shock Events

For visual drama, each ticker has a ~0.1% chance per tick of a sudden price shock:

```python
if random.random() < 0.001:  # ~0.1% per tick
    shock = random.uniform(0.02, 0.05)  # 2-5% magnitude
    direction = random.choice([-1, 1])
    price *= 1 + shock * direction
```

With 10 tickers at 2 ticks/second, expect roughly one event every 50 seconds. These create the sudden jumps that make watching the price stream feel alive.

## Seed Data

Defined in `seed_prices.py`:

### Starting Prices

```python
SEED_PRICES = {
    "AAPL": 190.00,  "GOOGL": 175.00, "MSFT": 420.00,
    "AMZN": 185.00,  "TSLA": 250.00,  "NVDA": 800.00,
    "META": 500.00,  "JPM": 195.00,   "V": 280.00,
    "NFLX": 600.00,
}
```

### Per-Ticker Parameters

| Ticker | Sigma (volatility) | Mu (drift) | Character |
|--------|-------------------|------------|-----------|
| AAPL | 0.22 | 0.05 | Moderate, steady |
| GOOGL | 0.25 | 0.05 | Moderate |
| MSFT | 0.20 | 0.05 | Lower volatility |
| AMZN | 0.28 | 0.05 | Slightly volatile |
| TSLA | 0.50 | 0.03 | Very volatile, low drift |
| NVDA | 0.40 | 0.08 | Volatile, strong upward drift |
| META | 0.30 | 0.05 | Moderate-high volatility |
| JPM | 0.18 | 0.04 | Stable (bank stock) |
| V | 0.17 | 0.04 | Stable (payments) |
| NFLX | 0.35 | 0.05 | Moderate-high volatility |

Tickers added dynamically at runtime (via watchlist) get default params: `sigma=0.25`, `mu=0.05`, with a random seed price between $50-$300.

## Code Structure

### GBMSimulator (`simulator.py`)

The core math engine. Stateful — holds current prices and the Cholesky matrix.

```python
class GBMSimulator:
    def __init__(self, tickers: list[str], dt: float, event_probability: float)
    def step(self) -> dict[str, float]       # Advance one tick, return new prices
    def add_ticker(self, ticker: str)         # Add + rebuild Cholesky
    def remove_ticker(self, ticker: str)      # Remove + rebuild Cholesky
    def get_price(self, ticker: str) -> float | None
    def get_tickers(self) -> list[str]
```

`step()` is the hot path — called every 500ms. It:
1. Generates N independent standard normal draws via `np.random.standard_normal(n)`
2. Applies Cholesky: `z_correlated = L @ z_independent`
3. For each ticker: applies GBM formula, checks for random event, rounds to 2 decimals
4. Returns `{ticker: new_price}` dict

### SimulatorDataSource (`simulator.py`)

Async wrapper that implements `MarketDataSource`. Bridges `GBMSimulator` to the `PriceCache`.

```python
class SimulatorDataSource(MarketDataSource):
    def __init__(self, price_cache: PriceCache, update_interval=0.5, event_probability=0.001)
```

- `start()`: creates `GBMSimulator`, seeds cache with initial prices, starts async loop
- Async loop: calls `simulator.step()`, writes each price to `cache.update()`, sleeps 500ms
- `add_ticker()`: delegates to simulator, seeds cache immediately
- `remove_ticker()`: delegates to simulator, removes from cache
- `stop()`: cancels the async task

### Dependencies

- `numpy` — for `np.random.standard_normal()`, `np.linalg.cholesky()`, and matrix multiplication
- `math` — for `exp()` and `sqrt()` in the GBM formula
- `random` — for shock events (probability check, magnitude, direction)

## Behavioral Characteristics

| Property | Value |
|----------|-------|
| Update frequency | Every 500ms |
| Price granularity | Rounded to 2 decimal places |
| Typical per-tick movement | Sub-cent (due to tiny dt) |
| Shock events | ~0.1% chance/tick/ticker, 2-5% magnitude |
| Shock frequency (10 tickers) | Roughly every 50 seconds |
| Correlation between tech stocks | 0.6 (visible as sector-wide moves) |
| Price range | Always positive (log-normal), never zero |

## Testing

17 unit tests for `GBMSimulator` and 10 integration tests for `SimulatorDataSource` cover:

- Prices always positive after many steps
- Statistical properties (mean return, volatility) are within expected bounds
- Correlation between tickers is in the expected direction
- Shock events occur at roughly the expected frequency
- Adding/removing tickers works correctly and rebuilds Cholesky
- SimulatorDataSource properly writes to PriceCache
