# QuantStand — System Architecture Reference
**Version:** 2.2  
**Date:** May 2026  
**Status:** Living document — update at end of every session  
**Repo:** github.com/QS-AlgoTrading

---

## Changelog v2.1 → v2.2

| # | Change |
|---|--------|
| 1 | Scope upgraded: QuantStand is now a family office platform, not a single-engine trading system |
| 2 | Three-engine model introduced: Engine 1 (futures) · Engine 2 (long-term allocation) · Engine 3 (arb, future) |
| 3 | Capital allocation layer added above all engines |
| 4 | Shared layers formalised: research (backtest + robustness) and infrastructure (data, connectors, audit) |
| 5 | Module structure revised to reflect platform layout — engine_futures/ · engine_allocation/ · engine_arb/ · shared/ · platform/ |
| 6 | Broker decisions locked: Binance Futures (crypto futures) · KuCoin Spot (crypto spot) · cTrader/FxPro (FX) · Interactive Brokers (equities) |
| 7 | Simulation adapters introduced in shared/research/ — one per engine type, not one per strategy |
| 8 | Blocker #9 (equities broker) resolved — Interactive Brokers selected |
| 9 | Engine 1 runtime flow and all locked decisions carried forward unchanged |

---

## What This Document Is

This document describes how QuantStand works — its platform scope, how engines are isolated, what is genuinely shared, and how data flows at runtime inside Engine 1 (the active futures engine). It is not a build sequence.

Read the Platform Model section first. Then read the Engine 1 Runtime Flow. Everything else supports those two sections.

---

## Platform Scope — Family Office Operating System

QuantStand is the central system for a family office: research, strategy development, backtesting, capital allocation, and execution across three market types. It is not a single-strategy quant platform. It is the platform that hosts multiple strategy engines.

**Three market access points:**

| Broker / Exchange | Market | Engine |
|-------------------|--------|--------|
| Binance Futures | Crypto futures | Engine 1 |
| KuCoin Spot | Crypto spot | Engine 2 |
| cTrader / FxPro | Forex (EURUSD · XAUUSD) | Engine 1 |
| Interactive Brokers | Equities | Engine 2 (future) |

**Long-term goal:** a portfolio of uncorrelated strategies across asset classes and strategy types — not single-strategy or single-engine dependence.

---

## Stack

- Language: Python
- Backtesting: VectorBT
- Optimisation: Optuna (Bayesian)
- UI: Streamlit
- Infrastructure: Self-hosted Mac Mini (all IP local, no cloud dependency)
- Also active: OpenClaw (AI chart vision agent for signals)

---

## Platform Model — Three Isolated Engines

QuantStand is a platform that hosts three engines. Each engine is capital-isolated. The platform layer sees P&L and drawdown per engine. It never sees inside an engine's risk logic.

```
┌─────────────────────────────────────────────────────────────────────┐
│  CAPITAL ALLOCATION LAYER                                           │
│  AUM tracking · per-engine allocation · rebalancing schedule        │
│  Cross-engine P&L + drawdown monitoring                             │
│  Hard rule: no cross-engine capital borrowing                       │
│  Hard rule: no cross-engine risk accounting                         │
└──────────┬────────────────────────┬────────────────────────┬────────┘
           │                        │                        │
           ▼                        ▼                        ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   ENGINE 1      │      │   ENGINE 2      │      │   ENGINE 3      │
│ Active futures  │      │ Long-term alloc │      │ Arbitrage       │
│                 │      │                 │      │ (future)        │
│ Crypto futures  │      │ DCA · rebal.    │      │ Stat-arb        │
│ Forex futures   │      │ Spot buy/sell   │      │ Funding arb     │
│                 │      │                 │      │ Cross-exchange  │
│ Regime detect.  │      │ Weight-based    │      │                 │
│ Strategy layer  │      │ allocation      │      │ Separate cap.   │
│ Portfolio constr│      │ construction    │      │ + risk model    │
│ Risk gate       │      │ Risk gate       │      │                 │
│                 │      │                 │      │                 │
│ Binance Futures │      │ KuCoin Spot     │      │ Multi-venue     │
│ cTrader/FxPro   │      │ IBKR (equities) │      │ routing         │
└─────────────────┘      └─────────────────┘      └─────────────────┘
           │                        │                        │
           └────────────────────────┴────────────────────────┘
                                    │
           ┌─────────────────────────────────────────────────┐
           │  SHARED RESEARCH LAYER                          │
           │  Backtest framework · IS/OOS · walk-forward     │
           │  Optuna optimiser · Bayesian search             │
           │  Robustness: permutation · Monte Carlo          │
           │  Pass/fail gate                                 │
           │  Simulation adapters (one per engine type)      │
           └─────────────────────────────────────────────────┘
                                    │
           ┌─────────────────────────────────────────────────┐
           │  SHARED INFRASTRUCTURE LAYER                    │
           │  Data pipelines · Parquet store                 │
           │  Exchange / broker connectors                   │
           │  Audit log · Streamlit UI · Reporter            │
           └─────────────────────────────────────────────────┘
```

### Capital Boundary Rules — Never Violate

- Each engine's capital allocation is decided at the platform level, transferred to the engine, and managed independently by that engine
- The platform layer receives P&L and drawdown per engine — it does not see inside engine risk logic
- Cross-engine capital borrowing is permanently prohibited — no exceptions
- Cross-engine risk accounting is permanently prohibited — Engine 2's spot holdings do not count toward Engine 1's correlation checks

### What Is Genuinely Shared

**Research layer — shared framework, engine-specific adapter:**

The backtest framework, optimiser, and robustness tests are identical regardless of strategy type. What differs is the simulation adapter — how VectorBT models a futures position vs a spot DCA vs an arb spread.

| Component | Shared? | Note |
|-----------|---------|------|
| Backtest framework (IS/OOS, walk-forward) | Yes | One implementation |
| Optuna optimiser | Yes | One implementation |
| Permutation test | Yes | Already built |
| Monte Carlo | Yes | Already built |
| Pass/fail gate | Yes | One implementation |
| Simulation adapters | No | One per engine type: futures · spot · arb |

**Infrastructure layer — pure I/O, zero business logic:**

| Component | Shared? |
|-----------|---------|
| Data fetchers (Binance, KuCoin, cTrader, IBKR, on-chain, COT) | Yes |
| Parquet store | Yes |
| Exchange / broker connectors | Yes |
| Audit log | Yes |
| Streamlit UI | Yes |
| Platform reporter | Yes |

### What Is NOT Shared

- Portfolio construction logic — each engine has its own, incompatible risk framework
- Risk gate — each engine defines its own rules and thresholds
- Position sizing — fundamentally different between futures (stop-based) and allocation (weight-based)
- RegimeConfig / StrategyConfig — Engine 1 owns these; Engine 2 will define its own equivalent config objects

---

## Engine 1 — Active Futures Engine

Engine 1 is the currently active build. It contains the full directional futures architecture: regime detection, strategy layer, portfolio construction, risk gate, and execution via Binance Futures and cTrader.

Everything in the task list (Tasks 1.1 through 2.4) is inside Engine 1.

### Runtime Flow — What Happens on a Signal Trigger

```
┌─────────────────────────────────────────────────────────────────────┐
│  SIGNAL TRIGGER (timeframe and trigger condition per strategy config)│
└──────────────┬──────────────────────┬───────────────────────────────┘
               │                      │
               ▼                      ▼
   ┌───────────────────┐  ┌───────────────────┐
   │   DATA LAYER      │  │   DATA LAYER      │     (one per asset class)
   │   Crypto          │  │   Forex           │  ···
   │   OHLCV · funding │  │   OHLCV · COT     │
   │   stablecoin      │  │   swap · CB cal.  │
   │   on-chain        │  │                   │
   └────────┬──────────┘  └────────┬──────────┘
            │                      │
     ┌──────┴──────┐        ┌──────┴──────┐
     ▼             ▼        ▼             ▼
 ┌────────┐  ┌─────────┐ ┌────────┐  ┌─────────┐
 │ REGIME │  │STRATEGY │ │ REGIME │  │STRATEGY │  ← run IN PARALLEL
 │DETECT. │  │    A    │ │DETECT. │  │    B    │    neither gates the other
 │        │  │         │ │        │  │         │
 │RVol+   │  │indicators│ │RVol+  │  │indicators│
 │Hurst   │  │per config│ │Hurst  │  │per config│
 │+suppl. │  │timeframe │ │+suppl.│  │timeframe │
 │inputs  │  │per config│ │inputs │  │per config│
 │        │  │         │ │        │  │         │
 │→ label │  │→ signal │ │→ label │  │→ signal │
 │(4-state│  │LONG /   │ │(4-state│  │LONG /   │
 │ enum)  │  │SHORT /  │ │ enum)  │  │SHORT /  │
 │        │  │FLAT     │ │        │  │FLAT     │
 └───┬────┘  └────┬────┘ └───┬────┘  └────┬────┘
     │             │          │             │
     └──────┬──────┘          └──────┬──────┘
            │                        │
            ▼                        ▼
┌───────────────────────────────────────────────────────────────────┐
│  PORTFOLIO CONSTRUCTION LAYER          ← FIRST point of interaction│
│                                                                    │
│  Reads all regime labels + all strategy signals simultaneously     │
│  Both are data inputs — neither gates the other upstream          │
│                                                                    │
│  Step 1 — Regime evaluation (strategy-configured logic)           │
│    Regime label is a data input, not a hardcoded gate             │
│    Each strategy declares its valid regime set in StrategyConfig  │
│    e.g. trend-following strategy: valid = [TRENDING]              │
│    e.g. mean-reversion strategy: valid = [RANGING]                │
│    e.g. multi-regime strategy: valid = [TRENDING, RANGING]        │
│    Regime label ∈ valid set → signal proceeds                     │
│    Regime label ∉ valid set → output flat for this strategy       │
│    Applied per strategy × per instrument independently            │
│                                                                    │
│  Step 2 — Position sizing (strategy + asset configured)           │
│    Sizing method is defined in StrategyConfig — not universal     │
│    Method and parameters differ per strategy and per asset class  │
│    Common risk denominator maintained across all active positions │
│    e.g. ATR-based: Size = (Equity × Risk%) ÷ ATR  [ARS/XAUUSD]  │
│    e.g. other methods: defined in strategy config at build time   │
│                                                                    │
│  Step 3 — Correlation check                                        │
│    Are active signals correlated? Cap total portfolio exposure     │
│    No doubling of risk across correlated instruments               │
│                                                                    │
│  Output: target portfolio state per instrument                     │
│    e.g. XAUUSD LONG · N lots · stop and size per strategy config  │
│         BTC/USDT FLAT · 0                                          │
└──────────────────────────────┬────────────────────────────────────┘
                               │
                               ▼
┌───────────────────────────────────────────────────────────────────┐
│  RISK GATE — hard rules, no exceptions                             │
│  Max drawdown breach → halt all trading                           │
│  Position limits → max size per asset class                       │
│  Human review gate → borderline cases flagged, you approve        │
└──────────────────────────────┬────────────────────────────────────┘
                               │
                               ▼
┌───────────────────────────────────────────────────────────────────┐
│  EXECUTION — orders only, no decisions                             │
│  Reads target portfolio state, submits only the delta orders      │
│  Binance Futures (crypto) · cTrader/FxPro (FX)                   │
│  Fill report fed back to portfolio state                          │
│  Streamlit UI · OpenClaw signals · risk monitor                   │
└───────────────────────────────────────────────────────────────────┘
```

### Critical Reading Notes

**Regime detection and strategy run in parallel.** They are siblings, not parent and child. Both read the same candle data independently. Neither knows the other exists. The portfolio construction layer is the first — and only — place they interact.

**Regime label is a data input, not a gate.** The regime detector's job ends at producing a label. It has no authority over whether a signal is acted on. That decision lives in the portfolio construction layer, which applies each strategy's configured valid-regime set.

**The portfolio construction layer is the decision brain.** It is the only place where regime state, signal validity, capital allocation, and correlation are resolved together. Every layer above it generates inputs. Every layer below it executes outputs.

**No strategy-specific values are baked into the architecture.** Timeframes, indicators, sizing methods, valid regime sets — all live in StrategyConfig and AssetConfig. The architecture is the contract. The config is the calibration.

**Execution knows nothing.** It receives a target portfolio state and submits the orders needed to reach it. It has no knowledge of regime, signals, or sizing rationale.

**Robustness testing is offline.** IS optimisation, Monte Carlo, permutation test, and walk-forward OOS run offline via the shared research layer. They feed calibrated parameters into the system but are not part of the live runtime path.

---

## Parameterised Framework — One Codebase, Per-Strategy and Per-Asset Calibration

The architectural framework inside Engine 1 is identical across all strategies and asset classes. What differs is the calibration held in config objects.

### What is shared within Engine 1 (one implementation)

- Regime detection algorithm: RVol dual-window + Hurst exponent + rule-based classifier
- 4-state regime model: TRENDING / RANGING / CHOPPY / TRANSITIONING
- Portfolio construction logic: regime evaluation → sizing → correlation check
- Clean architecture layer structure

### What is strategy-configured (StrategyConfig per strategy instance)

- Signal timeframe and entry refinement timeframe
- Indicators and their parameters
- Valid regime set: which regime labels permit signal execution
- Position sizing method and its parameters

### What is asset-configured (AssetConfig per instrument)

| Parameter | Crypto (BTC/ETH) | Forex (EURUSD/XAUUSD) | Equities |
|-----------|-----------------|----------------------|----------|
| RVol windows | fast=10 · slow=42 | TBD | TBD |
| Hurst window | 100 candles | TBD | TBD |
| IS anchor period | 2022–2024 | TBD | TBD |
| Supplementary inputs | Funding rate · stablecoin supply | COT · CB calendar · session vol | VIX · earnings calendar |
| Execution routing | Binance Futures | cTrader / FxPro | IBKR |

### In code

```python
# Regime config — one per instrument, generic from first build
RegimeConfig(
    asset="BTC/USDT",
    rvol_fast=10,
    rvol_slow=42,
    hurst_window=100,
    is_anchor="2022-01-01:2024-12-31",
    supplementary=["funding_rate", "stablecoin_supply"]
)

RegimeConfig(
    asset="XAUUSD",
    rvol_fast=TBD,
    rvol_slow=TBD,
    hurst_window=TBD,
    is_anchor=TBD,
    supplementary=["COT", "cb_calendar"]
)

# Strategy config — one per strategy instance
StrategyConfig(
    name="ARS",
    instrument="XAUUSD",
    signal_timeframe="4h",          # ARS-specific, not architecture-wide
    entry_timeframe="15m",          # ARS-specific, not architecture-wide
    valid_regimes=[RegimeLabel.TRENDING],
    sizing_method="ATR",            # ARS-specific, not architecture-wide
    sizing_params={"risk_pct": TBD, "atr_period": TBD}
)

# Different strategy, different config — architecture unchanged
StrategyConfig(
    name="StrategyB",
    instrument="BTC/USDT",
    signal_timeframe=TBD,
    entry_timeframe=TBD,
    valid_regimes=[RegimeLabel.RANGING],   # mean-reversion: valid in RANGING
    sizing_method=TBD,
    sizing_params={}
)
```

The `RegimeConfig` and `StrategyConfig` dataclasses must be designed to be generic from the first build. Retrofitting a single-strategy config to accept multi-strategy parameters means touching live production code. Design it correctly now.

---

## Layer Responsibilities — Engine 1

Each layer has one job. The "must not know" column is as important as the responsibility.

| Layer | Sole responsibility | Must NOT know |
|-------|-------------------|---------------|
| Data | Fetch, validate, store raw market data per asset class | Signals, regime, capital allocation |
| Regime detection | Classify market state → RegimeLabel per candle | Which strategies exist, position sizing, whether to trade |
| Strategy | Generate directional signals: Long / Short / Flat | Current regime, position size, other strategies' signals, whether its signal is acted on |
| Portfolio construction | Evaluate regime per strategy config · size positions per strategy + asset config · check correlation · output target portfolio state | How to submit orders, broker APIs, fill outcomes |
| Risk gate | Enforce hard rules: drawdown, position limits, human review | Signal logic, regime state, sizing rationale |
| Execution | Submit orders to reach target state, report fills | Why any position was chosen, anything above the risk gate |

---

## Module Structure — Full Platform

```
quantstand/
│
├── platform/                      ← family office level — no trading logic
│   ├── capital_allocator.py       (AUM tracking · per-engine allocation · rebalancing triggers)
│   ├── platform_risk_monitor.py   (cross-engine P&L · drawdown per engine · no engine internals)
│   └── platform_reporter.py       (consolidated reporting · allocation history)
│
├── shared/
│   │
│   ├── research/                  ← backtest + robustness — all engines call in
│   │   ├── backtest_framework.py  (IS/OOS splits · walk-forward · data split rules)
│   │   ├── optimiser.py           (Optuna wrapper · Bayesian search)
│   │   ├── robustness/
│   │   │   ├── permutation_test.py    (already built)
│   │   │   ├── monte_carlo.py         (already built)
│   │   │   └── pass_fail_gate.py      (Sharpe decay · sensitivity · trade count rules)
│   │   └── adapters/              ← strategy-type-specific simulation logic only
│   │       ├── futures_adapter.py (VectorBT — directional futures · stop-based risk)
│   │       ├── spot_adapter.py    (VectorBT — DCA · rebalancing · weight-based)
│   │       └── arb_adapter.py     (future)
│   │
│   ├── data/                      ← one fetcher per source · all engines read same Parquet store
│   │   ├── binance_fetcher.py     (REST + WS · OHLCV · futures)
│   │   ├── kucoin_fetcher.py      (REST + WS · OHLCV · spot · top 50 coins)
│   │   ├── ctrader_fetcher.py     (OHLCV · EURUSD · XAUUSD)
│   │   ├── ibkr_fetcher.py        (equities OHLCV · future)
│   │   ├── funding_rate_fetcher.py(Binance + KuCoin · 8h cycle)
│   │   ├── onchain_fetcher.py     (Glassnode · Nansen · stablecoin supply)
│   │   ├── cot_fetcher.py         (CFTC weekly · commercial vs speculative)
│   │   └── parquet_store.py       (read/write · shared across all engines)
│   │
│   ├── connectors/                ← order submission only · one per broker/exchange
│   │   ├── binance_connector.py   (Binance Futures · Engine 1)
│   │   ├── kucoin_connector.py    (KuCoin Spot · Engine 2)
│   │   ├── ctrader_connector.py   (FxPro / cTrader · EURUSD · XAUUSD)
│   │   └── ibkr_connector.py      (Interactive Brokers · equities · future)
│   │
│   └── audit/
│       └── audit_log.py           (append-only · every order · every fill · every capital movement)
│
├── engine_futures/                ← Engine 1 — active build
│   │
│   ├── domain/                    ← pure logic · zero I/O · fully unit-testable
│   │   ├── data/
│   │   │   └── data_models.py     (OHLCV, FundingRate, COT dataclasses)
│   │   ├── regime/
│   │   │   ├── regime_models.py   (RegimeLabel enum · RegimeConfig dataclass — generic)
│   │   │   ├── regime_features.py (realized_vol · hurst_exponent — pure functions)
│   │   │   └── regime_classifier.py (rule-based · hard gate → RegimeLabel)
│   │   ├── strategy/
│   │   │   ├── strategy_models.py (StrategyConfig · StrategySignal dataclasses — generic)
│   │   │   └── ars.py             (ARS signal logic — one file per strategy)
│   │   ├── portfolio/
│   │   │   ├── portfolio_models.py    (TargetPortfolioState · PositionTarget)
│   │   │   └── portfolio_construction.py (regime eval · sizing dispatch · correlation)
│   │   └── execution/
│   │       ├── order_models.py    (Order · Fill · Position)
│   │       └── risk_rules.py      (max drawdown · position limits)
│   │
│   ├── application/               ← orchestration only · no external deps
│   │   ├── regime_use_case.py     (load → features → classify → output Series)
│   │   ├── portfolio_use_case.py  (orchestrate: regime + signals → target state)
│   │   ├── backtest_use_case.py   (calls shared/research/ with futures_adapter)
│   │   └── execution_use_case.py  (paper → live gate · risk monitor)
│   │
│   └── infrastructure/            ← reads shared/ — no business logic
│       ├── parquet_loader.py      (loads from shared/data/parquet_store)
│       ├── ctrader_executor.py    (thin wrapper → shared/connectors/ctrader_connector)
│       ├── binance_executor.py    (thin wrapper → shared/connectors/binance_connector)
│       └── streamlit_ui.py        (UI layer — no business logic)
│
├── engine_allocation/             ← Engine 2 — design complete · build later
│   │
│   ├── domain/
│   │   ├── strategy/              (DCA rules · rebalancing logic · spot signal models)
│   │   ├── portfolio/             (weight-based construction · drift rules — NOT shared with Engine 1)
│   │   └── execution/             (drawdown tolerance · allocation risk rules)
│   │
│   ├── application/
│   │   └── backtest_use_case.py   (calls shared/research/ with spot_adapter)
│   │
│   └── infrastructure/            (reads shared/ — no business logic)
│
└── engine_arb/                    ← Engine 3 — placeholder only
    ├── domain/
    ├── application/
    └── infrastructure/
```

### Layer Enforcement Rules — Never Violate

| Rule | Detail |
|------|--------|
| `domain/` has zero imports from `infrastructure/` or `application/` | If a domain test needs a mock, a layer violation exists |
| `application/` imports `domain/` only | No Binance, no filesystem, no VectorBT in use cases |
| `infrastructure/` imports `application/` interfaces only | No business logic in infrastructure |
| All I/O lives in `infrastructure/` or `shared/` exclusively | No file reads, no API calls in domain or application |
| Dependency direction: infrastructure → application → domain | Never reverse |
| Engines read `shared/` — `shared/` never imports from engines | No reverse dependency through shared layers |
| `platform/` never imports engine internals | Only P&L and drawdown surface — no engine logic |
| `portfolio_construction.py` reads regime label as data input | Not as a module dependency — passed as a parameter |
| Each strategy is its own module in `domain/strategy/` | No shared strategy file — one file per strategy |
| Engine 1 and Engine 2 do NOT share portfolio construction | Fundamentally incompatible risk frameworks |

---

## Robustness Testing (Offline — Not Part of Live Runtime)

Every strategy must pass all gates before reaching live. This pipeline runs offline via `shared/research/`.

```
IS data → Optuna optimisation (Bayesian) → Validation OOS (walk-forward)
                                                      ↓
                              Permutation test (p < 0.05 required)
                              Monte Carlo 1000+ runs
                              pytest unit + integration suite
                                                      ↓
                              Auto pass/fail gate:
                              · OOS Sharpe ≥ 60% of IS
                              · p < 0.05
                              · OOS trade count > 5× free parameters (min 100)
                              · Sensitivity: ±10% on any param must not halve Sharpe
                                                      ↓
                              Human review gate → approved → paper trading → live
```

Three data splits — boundaries must be locked in repo before any optimisation run:
- IS: optimisation and parameter calibration
- Validation OOS: walk-forward, strategy selection
- True holdout: viewed exactly once, ever. Never used for selection.

The simulation adapter passed to the framework determines how trades are modelled. Engine 1 passes `futures_adapter`. Engine 2 will pass `spot_adapter`. The framework is identical.

---

## Crypto Data — Four-Layer Model (Engine 1)

Ranked by edge quality. Most retail traders only reach Layer 1.

**Layer 1 — Price and volume (competed · low edge)**
Use as inputs to features. Never as primary edge source.
- OHLCV — Binance REST + WS · Parquet · built
- Realized vol — computed from OHLCV · no new data needed · **regime feature (build now)**
- Hurst exponent — computed from OHLCV · H > 0.55 = trending · **regime feature (build now)**
- Open interest / liquidations — build later

**Layer 2 — Derivatives market structure (crypto-unique · medium-high edge)**
- Funding rate — perp vs spot premium · 8h cycle · Binance API · free · **build 1st**
- Basis / spread — contango / backwardation · build later
- Options flow — IV skew · put/call ratio · Deribit · build later
- Long/short ratio — retail positioning contrarian · build later

**Layer 3 — On-chain data (highest differentiation)**
- Stablecoin supply — USDT/USDC mint rate · new capital indicator · **build 3rd**
- Exchange inflows/outflows — Glassnode, Nansen · build later
- NUPL / SOPR — holder profit/loss · macro regime · build later
- Miner behaviour — Puell multiple, MRI · build later

**Layer 4 — Sentiment and social (noisy · extremes only)**
- Fear & Greed — regime label only, never entry trigger · build later
- Social volume — **deprioritised** · signal horizon mismatch
- News NLP — Messari, CryptoPanic · build later
- Google Trends — retail attention only · build later

**Key principle:** Layer 1 signals are inputs to features, not edge sources. Edge starts at Layer 2.

---

## Locked Decisions — Do Not Revisit Without Strong Evidence

| Decision | Value | Reason |
|----------|-------|--------|
| Platform model | Three isolated engines + shared layers | Capital and risk isolation required; shared infrastructure reduces duplication |
| Capital boundary | Hard — no cross-engine borrowing or risk accounting | Ambiguous risk accounting is how family offices get hurt |
| Engine 1 broker (crypto futures) | Binance Futures | Active |
| Engine 1 broker (FX) | cTrader / FxPro | Active |
| Engine 2 broker (crypto spot) | KuCoin Spot | Decided |
| Engine 2 broker (equities) | Interactive Brokers | Decided |
| Shared research layer | One framework · engine-specific simulation adapter | Prevents duplicating permutation/MC/optimiser code per engine |
| Portfolio construction | Not shared between engines | Engine 1 (stop-based) and Engine 2 (weight-based) are incompatible |
| Data splits | IS · Validation OOS · True holdout | Holdout viewed exactly once, ever |
| Regime classifier | Rule-based · hard gate | Interpretable, no ML overfit risk |
| Regime states | 4: TRENDING / RANGING / CHOPPY / TRANSITIONING | 4th state suppresses boundary whipsaws |
| Hurst window (crypto) | 100 candles — theory-fixed | Not optimised. Per-asset for other classes. |
| RVol windows (crypto) | fast=10 · slow=42 — theory-fixed | Not optimised. Per-asset for other classes. |
| RVol thresholds | IS-percentile anchored (2022–2024 for crypto), frozen | No drift, no retroactive relabelling |
| Regime placement | Parallel to strategy — NOT upstream of it | Both feed portfolio construction layer as data inputs |
| Regime label role | Data input to portfolio construction — not a gate | Gate logic lives in portfolio construction, configured per strategy |
| Portfolio construction layer | Dedicated layer between strategy and execution | Single responsibility: regime evaluation + sizing dispatch + correlation |
| Strategy config | Timeframes, indicators, valid regimes, sizing method all in StrategyConfig | Never baked into architecture — config is calibration, architecture is contract |
| Layer 1 signals | Inputs only — not edge sources | Competed. Edge starts at Layer 2. |
| F&G index | Regime label only — never entry trigger | Weak predictive power at non-extreme readings |
| Social volume | Deprioritised | Signal horizon mismatch for swing trading |
| RegimeConfig | Generic dataclass from first build | Prevents retrofitting when FX/equities added |
| StrategyConfig | Generic dataclass from first build | Prevents retrofitting when new strategies added |
| Long-term goal | Portfolio of uncorrelated strategies across engines | Not single-strategy or single-engine dependence |
| Infrastructure | Self-hosted Mac Mini | All IP stays local |

### ARS Strategy — Locked Decisions (ARS-specific, not architecture-wide)

| Decision | Value | Reason |
|----------|-------|--------|
| ARS signal timeframe | 4h | 1h tested and rejected — worse drawdown |
| ARS entry refinement | 15m on top of 4h signal | Does not change direction |
| ARS valid regimes | TRENDING only | Trend-following strategy |
| ARS sizing method | ATR-based | Calibrated to XAUUSD volatility profile |
| ARS free parameters | 2 (SlCoef, RR) — 12 combinations | Severe overfit found at higher param counts |

---

## Open Blockers — Must Resolve Before Next Build

| # | Blocker | Status |
|---|---------|--------|
| 1 | ARS combination count audit | ✅ Done — 1430 passes on ~335B space, severe overfit found |
| 2 | ARS IS vs OOS Sharpe decay | ✅ Done — decay to 4–6% of IS. Red. Reduced to 2 free params. |
| 3 | ARS free parameter list | ✅ Done — 2 params (SlCoef, RR), 12 combinations |
| 4 | Data split date boundaries committed to repo | 🔴 OPEN — blocks any new optimisation run |
| 5 | Regime classifier design decisions | ✅ Done — all 5 confirmed |
| 6 | Regime architecture: macro + per-asset-class tiers? | 🔴 OPEN — must decide before extending regime beyond crypto |
| 7 | Common risk denominator for cross-asset sizing | 🔴 OPEN — must decide before portfolio construction layer design |
| 8 | Correlation check method for portfolio construction | 🔴 OPEN — must decide before portfolio construction layer design |
| 9 | Equities broker / API selection | ✅ Done — Interactive Brokers selected |

---

## Overfitting Rules — Apply to Every Strategy in Every Engine

1. Hypothesis first — write WHY before optimising any parameter
2. Minimise free parameters — each one is a liability
3. 5× rule — OOS trades > 5× number of free parameters (min 100)
4. Stability over peak — centre of performance plateau, not the spike
5. Three-split rule — IS · Validation OOS · True Holdout (holdout viewed exactly once)
6. Search deflation — >50 combinations: treat Sharpe skeptically · >200: true edge ≈ measured / 2
7. IS/OOS decay — OOS Sharpe < 60% of IS = red flag · < 30% = random noise
8. Sensitivity check — ±10% on any parameter must NOT halve Sharpe

---

*Update this document at the end of every session and re-upload to the Claude project.*
