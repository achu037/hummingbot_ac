# Codebase Overview

This document describes how this repository is organized and how its main pieces fit together.

## What this is

[Hummingbot](https://hummingbot.org) is an open-source framework for building and running automated crypto trading bots (market making, arbitrage, directional trading, etc.) across centralized and decentralized exchanges.

- **Language:** Python 3.10, with performance-critical parts written in Cython (`.pyx`/`.pxd`) and a little C++.
- **Version:** `2.3.0` (see `hummingbot/VERSION`).
- **License:** Apache 2.0 (see `LICENSE`).

## Top-level layout

| Path | Purpose |
|---|---|
| `bin/` | Entry points: `hummingbot.py` (interactive CLI), `hummingbot_quickstart.py` (headless / auto-start), `conf_migration_script.py` |
| `hummingbot/` | The main Python/Cython package (see below) |
| `controllers/` | Strategy V2 controllers, grouped into `directional_trading/`, `market_making/`, `generic/` |
| `scripts/` | Script strategies (single-file bots) — `basic/`, `community/`, `gateway/`, `utility/`, plus `simple_pmm.py`, `v2_with_controllers.py`, etc. |
| `conf/` | Runtime configuration (connectors, strategies, controllers, scripts) — mostly gitignored, created at runtime |
| `test/` | Unit tests; `test/hummingbot/` mirrors the package layout |
| `setup/` | Conda environment files (`environment.yml`, `environment_dydx.yml`) |
| `.github/` | CI workflows, composite install action, issue/PR templates |
| `setup.py` / `pyproject.toml` | Package metadata and Cython build configuration; black/isort settings |
| `Makefile` | `install`, `build`, `test`, `run_coverage`, `docker`, `clean`, … |
| `install`, `uninstall`, `compile`, `compile.bat`, `clean` | Shell scripts for environment setup and Cython compilation |
| `Dockerfile`, `docker-compose.yml` | Container image and runtime setup |
| `.pre-commit-config.yaml`, `.flake8`, `.coveragerc` | Linting and coverage configuration |

## How the runtime fits together

```
bin/hummingbot.py  /  bin/hummingbot_quickstart.py
        │
        ▼
HummingbotApplication  (hummingbot/client/hummingbot_application.py)
  - CLI UI, config loading, command handling (start, stop, connect, balance, …)
        │  `start` builds connectors + strategy, then:
        ▼
Clock  (hummingbot/core/clock.pyx; REALTIME or BACKTEST mode)
  - ticks every registered TimeIterator once per tick interval
        │
        ├──► Connectors (ExchangePyBase subclasses) ── REST/WebSocket ──► exchanges
        │       │ order book trackers, user stream trackers, in-flight order tracking
        │       │
        │       └── emit events (fills, order created/cancelled, …) via PubSub
        │                         │
        └──► Strategy  ◄──────────┘
                (legacy StrategyBase, ScriptStrategyBase, or StrategyV2Base)
```

Supporting services: `MarketsRecorder` (`hummingbot/connector/markets_recorder.py`) persists orders and fills to SQLite via the SQLAlchemy models in `hummingbot/model/`; the rate oracle (`hummingbot/core/rate_oracle/`) provides conversion rates; the optional MQTT interface (`hummingbot/remote_iface/`) allows remote control.

## The `hummingbot/` package

| Subpackage | Purpose | Key files / classes |
|---|---|---|
| `client/` | CLI application | `hummingbot_application.py` (`HummingbotApplication`, composed of command mixins); `command/` (one mixin per CLI command); `config/` (`client_config_map.py`, `config_helpers.py`, `security.py`, `config_crypt.py`); `ui/` (prompt_toolkit layout, completer, parser); `settings.py` (`AllConnectorSettings`, paths) |
| `core/` | Runtime engine and shared primitives | `clock.pyx` (`Clock`), `time_iterator.pyx`, `network_iterator.pyx`, `pubsub.pyx`; `data_type/` (`OrderBook`, `LimitOrder`, `InFlightOrder`, order book / user stream trackers, `TradeFee`); `event/` (event types and listeners); `api_throttler/` (`AsyncThrottler`); `web_assistant/` (REST/WS assistants); `rate_oracle/`; `gateway/` (Gateway HTTP client); `cpp/` (C++ order book entries, limit orders) |
| `connector/` | Exchange connectors and their base classes | `connector_base.pyx` → `exchange_base.pyx` → `exchange_py_base.py` (`ExchangePyBase`, the base for modern connectors); `perpetual_derivative_py_base.py`; `client_order_tracker.py`, `budget_checker.py`, `time_synchronizer.py`, `trading_rule.pyx`, `markets_recorder.py` |
| `strategy/` | Legacy strategies and strategy base classes | `strategy_base.pyx`, `strategy_py_base.pyx`, `script_strategy_base.py` (`ScriptStrategyBase`), `strategy_v2_base.py` (`StrategyV2Base`), `market_trading_pair_tuple.py`, asset price delegates |
| `strategy_v2/` | V2 strategy framework | `controllers/` (`ControllerBase` and directional / market-making bases), `executors/` (`ExecutorBase`, `ExecutorOrchestrator`, concrete executors), `models/`, `backtesting/`, `runnable_base.py` |
| `data_feed/` | Market data | `market_data_provider.py` (`MarketDataProvider`), `candles_feed/` (per-exchange candle feeds + factory), `liquidations_feed/`, CoinGecko / CoinCap feeds, AMM gateway feed |
| `model/` | Persistence | SQLAlchemy models (`order.py`, `trade_fill.py`, `position.py`, `executors.py`, `controllers.py`, …), `sql_connection_manager.py` |
| `remote_iface/` | Remote control | `mqtt.py` (MQTT gateway, commands, notifier), `messages.py` |
| `logger/` | Logging | `logger.py`, `struct_logger.py`, CLI log handler |
| `notifier/` | Notifications | `notifier_base.py`, `telegram_notifier.py` |
| `templates/` | YAML templates | `conf_*_strategy_TEMPLATE.yml`, logging and fee override templates |
| `user/` | User balance helpers | `user_balances.py` |

> Note: `hummingbot/README.md` contains an older directory tree that predates `strategy_v2/`; prefer this document.

## Connectors (`hummingbot/connector/`)

- **`exchange/`** — spot exchange connectors (~26, e.g. `binance`, `bybit`, `okx`, `kucoin`, `gate_io`, `kraken`, `hyperliquid`, `xrpl`, `injective_v2`) plus `paper_trade` (simulated trading).
- **`derivative/`** — perpetual futures connectors (`binance_perpetual`, `bybit_perpetual`, `okx_perpetual`, `hyperliquid_perpetual`, `dydx_v4_perpetual`, …), plus shared `position.py` and `perpetual_budget_checker.py`.
- **`gateway/`** — DEX/AMM connectors that talk to the separate [Hummingbot Gateway](https://github.com/hummingbot/gateway) service (`amm/gateway_ethereum_amm.py`, `amm/gateway_solana_amm.py`, …).
- **`utilities/oms_connector/`** — shared base for OMS-style exchanges.
- **`test_support/`** — reusable test harnesses for connectors (`exchange_connector_test.py`, `perpetual_derivative_test.py`, `network_mocking_assistant.py`).

A typical connector folder (e.g. `hummingbot/connector/exchange/binance/`):

| File | Role |
|---|---|
| `binance_exchange.py` | Main connector class (subclass of `ExchangePyBase`); order placement, cancellation, balance and order status updates |
| `binance_api_order_book_data_source.py` | Public market data: order book snapshots/diffs, trades |
| `binance_api_user_stream_data_source.py` | Private WebSocket stream: order and balance updates |
| `binance_auth.py` | Request signing |
| `binance_constants.py` | URLs, endpoints, rate limits, order state mapping |
| `binance_web_utils.py` | URL builders and web assistant factory |
| `binance_utils.py` | Config map (API keys), fee defaults, helpers |
| `binance_order_book.py` | Message parsing into order book messages |
| `dummy.pyx` / `dummy.pxd` | Placeholder so the folder is picked up by the Cython build |

Derivative connectors follow the same pattern with `*_derivative.py` (subclassing `PerpetualDerivativePyBase`).

## Strategies

There are three generations of strategy APIs, all of which still work:

1. **Legacy strategies** — `hummingbot/strategy/<name>/` (`pure_market_making`, `avellaneda_market_making`, `cross_exchange_market_making`, `amm_arb`, `perpetual_market_making`, `spot_perpetual_arbitrage`, `liquidity_mining`, `hedge`, `twap`, `cross_exchange_mining`). Each has a `start.py`, a config map, and the implementation (some in Cython). Configured via the `create` command and YAML templates in `hummingbot/templates/`.
2. **Script strategies** — single Python files in `scripts/` subclassing `ScriptStrategyBase` (`hummingbot/strategy/script_strategy_base.py`). Implement `on_tick()` and event handlers; optionally take a pydantic config. Started with `start --script <file>.py`.
3. **Strategy V2** — a composable framework:
   - A script subclasses `StrategyV2Base` (`hummingbot/strategy/strategy_v2_base.py`), e.g. `scripts/v2_with_controllers.py`.
   - It loads one or more **controllers** (`controllers/` at the repo root, built on `hummingbot/strategy_v2/controllers/`). Controllers consume market data (candles, order books via `MarketDataProvider`) and emit `ExecutorAction`s.
   - The `ExecutorOrchestrator` (`hummingbot/strategy_v2/executors/executor_orchestrator.py`) runs **executors** that manage the order lifecycle: `position_executor`, `dca_executor`, `grid_executor`, `twap_executor`, `xemm_executor`, `arbitrage_executor`.
   - `hummingbot/strategy_v2/backtesting/` provides a backtesting engine for controllers.

## Cython

- Cython sources live mainly in `hummingbot/core/` (clock, pubsub, iterators, order book, events), `hummingbot/connector/` (base classes, paper trade), and `hummingbot/strategy/` (base classes, `pure_market_making`, `avellaneda_market_making`, `cross_exchange_mining`).
- `setup.py` cythonizes `hummingbot/**/*.pyx` as C++ (`language_level=3`).
- Environment flags: `WITHOUT_CYTHON_OPTIMIZATIONS` builds with `-O0` (faster compile, easier debugging); `DEV_MODE` appends `.dev1` to the version.
- After changing any `.pyx`/`.pxd` file, recompile with `./compile` (or `make build`).

## Setup, build, and run

```bash
make install            # ./install: creates/updates the `hummingbot` conda env from setup/environment.yml
conda activate hummingbot
make build              # ./compile: python setup.py build_ext --inplace
./bin/hummingbot.py     # start the interactive CLI
```

Headless start (e.g. in Docker):

```bash
./bin/hummingbot_quickstart.py -p <password> -f <strategy_config.yml>   # or --script-conf / -c for scripts
```

Docker:

- `Dockerfile` — multi-stage build on `continuumio/miniconda3`; default command runs `bin/hummingbot_quickstart.py`.
- `docker-compose.yml` — runs `hummingbot/hummingbot:latest`, mounting `conf/`, `logs/`, `data/`, `certs/`, `scripts/`, `controllers/`; includes a commented-out Gateway service.
- `make docker` builds the image locally.

## Tests and quality checks

- Tests live in `test/`, with `test/hummingbot/` mirroring the package structure. Shared helpers: `test/isolated_asyncio_wrapper_test_case.py`, `test/logger_mixin_for_test.py`.
- `make test` runs the suite with `coverage run -m nose` (some directories, e.g. gateway AMM and a few connectors, are excluded — see `Makefile`).
- `make run_coverage` produces coverage reports. `.coveragerc` enforces 70% overall; CI additionally requires 80% coverage on changed lines via `diff-cover`.
- `pre-commit` (installed by `./install`) runs flake8 (max line length 120), isort, and private-key detection.
- CI: `.github/workflows/workflow.yml` (lint + tests + coverage) and `.github/workflows/docker_buildx_workflow.yml` (image build).

## Known quirks

- Dependency pins are not fully consistent:
  - numpy is `1.23.5` in `setup/environment.yml` but `1.26.4` in `setup.py` and `pyproject.toml`.
  - xrpl-py is `3.0.0` in `setup/environment.yml` but `4.0.0b3` in `setup.py`.
  - Cython is `3.0.0` in `setup.py` but `3.0.0a10` in `pyproject.toml`.
- `setup.py` uses a date-style version (`20250203`) while `hummingbot/VERSION` is `2.3.0`.
- `hummingbot/README.md` describes an outdated directory layout.
