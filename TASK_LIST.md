# QuantStand — Task List
**Version:** 2.0
**Date:** May 2026
**Status:** Living document — update at end of every session
**Repo:** github.com/QS-AlgoTrading

---

## Priority 1 — Forex · XAUUSD

> Build sequence: Regime Detection → Strategy + Regime Integration → Allocation Layer → cTrader Execution

---

### 1.1 · Regime Detection — XAUUSD
**Status:** 🔴 Not started

Build the regime detection module for Gold using the parameterised framework.

- [ ] Create `RegimeConfig(asset="XAUUSD")` — calibrate RVol windows, Hurst window, IS thresholds on Gold data
- [ ] Define supplementary inputs: COT positioning + CB calendar events
- [ ] Implement `regime_features.py` for XAUUSD — realized vol, Hurst exponent (pure functions, zero I/O)
- [ ] Implement `regime_classifier.py` — rule-based hard gate → TRENDING / RANGING / CHOPPY / TRANSITIONING
- [ ] Wire data source: `ctrader_fetcher.py` → OHLCV for XAUUSD
- [ ] Unit tests: feature functions on Gold OHLCV
- [ ] Integration test: full regime label series output

---

### 1.2 · Strategy + Regime Integration — XAUUSD
**Status:** 🔴 Not started
**Depends on:** 1.1

Connect ARS signal output and regime label into the portfolio construction layer for Gold.

- [ ] Port ARS signal logic to XAUUSD instrument — 4h direction, 15m entry
- [ ] Confirm regime and strategy run in parallel — neither gates the other directly
- [ ] Portfolio construction layer reads both: regime label + ARS signal simultaneously
- [ ] Regime gate applied: TRENDING → signal valid · RANGING / CHOPPY / TRANS → output flat
- [ ] Unit tests: gate logic for all four regime states

---

### 1.3 · Allocation Layer — XAUUSD
**Status:** 🔴 Not started
**Depends on:** 1.2
**Blockers:** Common risk denominator decision (#7) · Correlation method decision (#8)

Design and build the position sizing and correlation rules for Gold within the portfolio construction layer.

- [ ] Resolve Blocker #7: common risk denominator across Crypto + Forex
- [ ] Resolve Blocker #8: correlation check method (EURUSD + XAUUSD partial USD correlation)
- [ ] Define XAUUSD sizing parameters: Risk% per trade, ATR period, lot size (pip-value normalised)
- [ ] Encode USD correlation rule: EURUSD + XAUUSD both active → cap total USD exposure
- [ ] `TargetPortfolioState` output defined: instrument · direction · size · stop
- [ ] Unit tests: sizing formula, correlation cap, portfolio state output

---

### 1.4 · cTrader Execution — XAUUSD
**Status:** 🔴 Not started
**Depends on:** 1.3

Wire the portfolio construction output to cTrader order submission for XAUUSD.

- [ ] `infrastructure/execution/ctrader_executor.py` — reads `TargetPortfolioState`, submits delta orders
- [ ] Order types confirmed: market, limit, stop-loss
- [ ] Fill report fed back to portfolio state
- [ ] Risk gate wired: max drawdown breach → halt, position limits enforced
- [ ] Human review gate: borderline cases flagged before live
- [ ] Paper trading run before any live capital

---

## Priority 2 — Crypto · 50 Coins

> Build sequence: KuCoin Data Layer → Scanner → Arshia Strategy on 50 Coins → Extended Data Layer

---

### 2.1 · KuCoin Connection + Data Layer
**Status:** 🔴 Not started

Add KuCoin as a data source and build the OHLCV pipeline for top 50 coins.

- [ ] Define coin selection criteria: top 50 by liquidity/volume (document the rule)
- [ ] `infrastructure/data/kucoin_fetcher.py` — REST + WS, OHLCV for top 50
- [ ] Parquet store extended for KuCoin instruments (same pattern as Binance)
- [ ] Funding rate fetcher: KuCoin + Binance, 8h cycle
- [ ] Unit tests: fetcher, Parquet read/write

---

### 2.2 · Lemmi-Style Scanner — Top 50 Coins
**Status:** 🔴 Not started
**Depends on:** 2.1

Regime detection across 50 coins, ranked by tradeable state.

- [ ] `RegimeConfig` instances for each coin — shared crypto config initially, document the decision
- [ ] Run regime detection across all 50 coins on each 4h candle close
- [ ] Scanner output: ranked list by regime state (TRENDING prioritised) + Arshia Strategy signal per coin
- [ ] Integration test: scanner output format validated

---

### 2.3 · Arshia Strategy on 50 Coins — Execution Ready
**Status:** 🔴 Not started
**Depends on:** 2.2

Connect Arshia Strategy (Arshia-Breakout) signal + regime output into allocation and execution for the 50-coin universe.

- [ ] Portfolio construction layer extended: handles N instruments, not just BTC/ETH
- [ ] Correlation check across 50 coins: cap concentrated crypto exposure
- [ ] `TargetPortfolioState` output per active instrument
- [ ] Execution wired: Binance Futures for top coins, KuCoin for remainder (define routing rule)
- [ ] Risk gate: per-instrument position limits, aggregate crypto exposure cap

---

### 2.4 · Extended Crypto Data Layer
**Status:** 🔴 Not started — phased build

Build additional data inputs in priority order.

**Phase A — Derivatives (build first)**
- [ ] Funding rate integrated into RegimeConfig as supplementary crowding signal
- [ ] Open interest fetcher — Binance + KuCoin

**Phase B — Order Flow + Structure (build after Phase A)**
- [ ] Block order / large limit order detection via order book depth
- [ ] Support / resistance level algorithm — define method before build (swing high/low, volume profile, or both)
- [ ] Volume profile per instrument: high-volume nodes as S/R input

**Phase C — On-Chain (build later)**
- [ ] Stablecoin supply (USDT/USDC mint rate): new capital indicator
- [ ] Exchange inflows/outflows: Glassnode or Nansen API

---

## Open Blockers

| # | Blocker | Blocks |
|---|---------|--------|
| 7 | Common risk denominator for cross-asset sizing | 1.3 |
| 8 | Correlation check method for portfolio construction | 1.3 |
| 9 | Equities broker / API selection | Future |

---

## Clean Architecture — Enforce on Every Task

| Layer | Rule |
|-------|------|
| `domain/` | Zero I/O · zero external imports · fully unit-testable |
| `application/` | Orchestration only · no Binance, no filesystem, no VectorBT |
| `infrastructure/` | All I/O lives here · no business logic |
| Dependency direction | infrastructure → application → domain · never reverse |

---

*Update this document at the end of every session and re-upload to the Claude project.*
