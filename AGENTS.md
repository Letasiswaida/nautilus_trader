# AGENTS.md

This file provides guidance to AI agents (including GitHub Copilot, Claude, and other LLMs) when working with NautilusTrader Python code. Follow these guidelines to ensure code adheres to project conventions, best practices, and the official API.

## Documentation Reference

Always consult the official NautilusTrader documentation at <https://nautilustrader.io/docs/> for the most up-to-date API reference, concepts, and examples.

## Project Overview

NautilusTrader is a high-performance algorithmic trading platform written primarily in Rust with Python bindings. It supports:

- **Backtesting**: Historical data with simulated venues
- **Sandbox**: Real-time data with simulated venues
- **Live trading**: Real-time data with live venues

## Python Code Style

### PEP-8 Compliance

Follow PEP-8 style guidelines with the following notes:

- Use explicit `None` checks (`if foo is None:`) rather than truthiness for non-collection types
- Use truthiness to check for empty collections (`if not my_list:`)

### Type Annotations

All functions and methods must include comprehensive type annotations:

```python
def __init__(self, config: EMACrossConfig) -> None:
def on_bar(self, bar: Bar) -> None:
def on_save(self) -> dict[str, bytes]:
```

Use PEP 604 union syntax for optional types:

```python
# Preferred
def get_instrument(self, id: InstrumentId) -> Instrument | None:

# Avoid
def get_instrument(self, id: InstrumentId) -> Optional[Instrument]:
```

### Docstrings

Use NumPy docstring format. Python docstrings should be written in the **imperative mood** (e.g., "Return a cached client.").

Do not add docstrings to private methods (prefixed with `_`) unless they have complex logic that requires explanation.

### Formatting

- Lines should generally stay below 100 characters
- For longer lines with multiple arguments, use a new line at the next logical indent:

```python
long_method_with_many_params(
    some_arg1,
    some_arg2,
    some_arg3,  # <-- trailing comma
)
```

## Strategy Development

### Strategy Structure

Strategies inherit from the `Strategy` class and optionally use a `StrategyConfig` for configuration:

```python
from nautilus_trader.config import StrategyConfig
from nautilus_trader.trading.strategy import Strategy

class MyStrategyConfig(StrategyConfig, frozen=True):
    instrument_id: InstrumentId
    bar_type: BarType
    trade_size: Decimal

class MyStrategy(Strategy):
    def __init__(self, config: MyStrategyConfig) -> None:
        super().__init__(config)  # Always initialize the parent class
```

### Lifecycle Handlers

Implement these handlers based on strategy needs:

| Handler | Purpose |
|---------|---------|
| `on_start()` | Initialize strategy: fetch instruments, subscribe to data, register indicators |
| `on_stop()` | Cleanup: cancel orders, close positions, unsubscribe from data |
| `on_reset()` | Reset indicators and internal state (between backtest runs) |
| `on_save()` | Return state dictionary for persistence |
| `on_load()` | Restore state from saved dictionary |

### Important Warnings

1. **Do not call `clock` or `logger` in `__init__`** - The system clock and logging subsystem are not initialized until after registration
2. **Access configuration via `self.config`** - Separates configuration from strategy state
3. **Use unique `order_id_tag`** - Required when running multiple strategy instances

### Data Handling

#### Historical vs Real-time Data

- **Historical data** (from requests like `request_bars()`): Processed by `on_historical_data()`
- **Real-time data** (from subscriptions like `subscribe_bars()`): Processed by specific handlers like `on_bar()`

```python
def on_start(self) -> None:
    # Request historical data - goes to on_historical_data()
    self.request_bars(self.bar_type)
    
    # Subscribe to real-time data - goes to on_bar()
    self.subscribe_bars(self.bar_type)

def on_historical_data(self, data: Data) -> None:
    # Handle historical data (from requests)
    pass

def on_bar(self, bar: Bar) -> None:
    # Handle real-time bar updates (from subscriptions)
    pass
```

### Order Management

Use the built-in `OrderFactory` available on every strategy:

```python
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.enums import TimeInForce

order = self.order_factory.market(
    instrument_id=self.instrument_id,
    order_side=OrderSide.BUY,
    quantity=self.instrument.make_qty(self.trade_size),
    time_in_force=TimeInForce.GTC,
)
self.submit_order(order)
```

### Cache and Portfolio Access

```python
# Cache access for data and execution objects
last_quote = self.cache.quote_tick(self.instrument_id)
last_bar = self.cache.bar(bar_type)
order = self.cache.order(client_order_id)
position = self.cache.position(position_id)

# Portfolio access for account information
is_flat = self.portfolio.is_flat(self.instrument_id)
is_long = self.portfolio.is_net_long(self.instrument_id)
unrealized_pnl = self.portfolio.unrealized_pnl(self.instrument_id)
```

### Indicators

Register indicators to receive automatic updates:

```python
def on_start(self) -> None:
    self.fast_ema = ExponentialMovingAverage(10)
    self.slow_ema = ExponentialMovingAverage(20)
    
    # Register indicators for automatic bar updates
    self.register_indicator_for_bars(self.bar_type, self.fast_ema)
    self.register_indicator_for_bars(self.bar_type, self.slow_ema)

def on_bar(self, bar: Bar) -> None:
    # Check if indicators are ready before using
    if not self.indicators_initialized():
        return
    
    # Now safe to use indicator values
    if self.fast_ema.value > self.slow_ema.value:
        # Trading logic
        pass
```

## Actor Development

Actors provide data handling without order management. Use for monitoring, data processing, or custom analysis:

```python
from nautilus_trader.common.actor import Actor
from nautilus_trader.config import ActorConfig

class MyActorConfig(ActorConfig):
    instrument_id: InstrumentId
    bar_type: BarType

class MyActor(Actor):
    def __init__(self, config: MyActorConfig) -> None:
        super().__init__(config)

    def on_start(self) -> None:
        self.subscribe_bars(self.config.bar_type)

    def on_bar(self, bar: Bar) -> None:
        self.log.info(f"Received bar: {bar}")
```

## Timers and Alerts

```python
import pandas as pd

def on_start(self) -> None:
    # Set recurring timer
    self.clock.set_timer(
        name="my_timer",
        interval=pd.Timedelta(minutes=1),
    )
    
    # Set one-time alert
    self.clock.set_time_alert(
        name="my_alert",
        alert_time=self.clock.utc_now() + pd.Timedelta(hours=1),
    )

def on_stop(self) -> None:
    # Always cancel timers on stop
    self.clock.cancel_timer("my_timer")
```

## Common Imports

```python
# Configuration
from nautilus_trader.config import StrategyConfig
from nautilus_trader.config import ActorConfig

# Strategy and Actor base classes
from nautilus_trader.trading.strategy import Strategy
from nautilus_trader.common.actor import Actor

# Data types
from nautilus_trader.model import Bar, BarType
from nautilus_trader.model import QuoteTick, TradeTick
from nautilus_trader.model import OrderBook, OrderBookDeltas

# Identifiers
from nautilus_trader.model import InstrumentId
from nautilus_trader.model import Venue

# Order types and enums
from nautilus_trader.model.enums import OrderSide
from nautilus_trader.model.enums import TimeInForce
from nautilus_trader.model.enums import TriggerType
from nautilus_trader.model.orders import MarketOrder, LimitOrder

# Quantities and prices
from nautilus_trader.model import Quantity, Price

# Indicators
from nautilus_trader.indicators import ExponentialMovingAverage

# Events
from nautilus_trader.model.events import OrderFilled
from nautilus_trader.model.events import PositionOpened, PositionClosed

# Logging
from nautilus_trader.common.enums import LogColor
```

## Best Practices

1. **Always check for `None` when fetching from cache**:
   ```python
   instrument = self.cache.instrument(self.instrument_id)
   if instrument is None:
       self.log.error(f"Could not find instrument for {self.instrument_id}")
       self.stop()
       return
   ```

2. **Use `self.instrument.make_qty()` and `self.instrument.make_price()`** to ensure correct precision:
   ```python
   quantity = self.instrument.make_qty(self.trade_size)
   price = self.instrument.make_price(100.50)
   ```

3. **Check indicator readiness before trading**:
   ```python
   if not self.indicators_initialized():
       self.log.info("Waiting for indicators to warm up")
       return
   ```

4. **Clean up on stop**:
   ```python
   def on_stop(self) -> None:
       self.cancel_all_orders(self.instrument_id)
       self.close_all_positions(self.instrument_id)
       self.unsubscribe_bars(self.bar_type)
   ```

5. **Handle order and position events appropriately**:
   ```python
   def on_order_filled(self, event: OrderFilled) -> None:
       self.log.info(f"Order filled: {event.order_side} {event.last_qty} @ {event.last_px}")
   
   def on_order_rejected(self, event: OrderRejected) -> None:
       self.log.warning(f"Order rejected: {event.reason}")
   ```

6. **Use configuration for strategy parameters** instead of hardcoding values:
   ```python
   class MyStrategyConfig(StrategyConfig, frozen=True):
       fast_ema_period: int = 10
       slow_ema_period: int = 20
       trade_size: Decimal
   ```

## Testing

Use descriptive test names that explain the scenario:

```python
def test_currency_with_negative_precision_raises_overflow_error(self):
def test_sma_with_no_inputs_returns_zero_count(self):
def test_strategy_submits_order_when_ema_crosses(self):
```

## Code Linting

The codebase uses `ruff` for linting. Run `ruff check` to verify code style compliance. Configuration is in `pyproject.toml`.

## Examples

Refer to the `examples/` directory for working examples organized by environment:

- `examples/backtest/` - Backtesting examples
- `examples/live/` - Live trading examples
- `examples/sandbox/` - Sandbox examples
- `nautilus_trader/examples/strategies/` - Example strategy implementations

## Additional Resources

- [Getting Started Guide](https://nautilustrader.io/docs/latest/getting_started/)
- [Concepts Documentation](https://nautilustrader.io/docs/latest/concepts/)
- [API Reference](https://nautilustrader.io/docs/latest/api_reference/)
- [Developer Guide](https://nautilustrader.io/docs/latest/developer_guide/)
