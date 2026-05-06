# QuantStand — Task List
_Last updated: 2026-05-06 (post ARS audit session)_

---

## Priority 1 — IMMEDIATE (this sprint)

### 1a. Regime Detection Module — DESIGN SESSION NEXT
- [ ] Design doc first (rule-based vs ML decision, features, states, output schema)
- [ ] Features: Realized vol · Hurst exponent · Funding rate · F&G filter
- [ ] States: high-vol trending / low-vol ranging / high-vol choppy
- [ ] Output: regime label per candle → gates all strategies
- [ ] Clean architecture: domain/ layer only, zero I/O
- [ ] Unit tests required before any integration

### 1b. Add Sortino Ratio to QuantStand Analytics
- [ ] Check existing Sharpe implementation in QS (verify in next session with repo access)
- [ ] Add `sortino_ratio()` alongside `sharpe_ratio()` — same input signature
- [ ] Add both to backtest report output
- [ ] Unit test: verify Sortino >= Sharpe when wins > losses in magnitude

### 1c. CoinGlass Data Layer — HIGH PRIORITY
- [ ] Design doc: what data, what endpoints, what cadence
- [ ] Liquidation heatmap endpoint (aggregated) — historical + live
- [ ] Liquidity/order book heatmap endpoint
- [ ] Store as Parquet alongside OHLCV data
- [ ] Hypothesis to test: do price-to-liquidation-cluster proximity levels predict bounce/continuation vs random S/R?
- [ ] Requires CoinGlass API key (paid tier for historical data)
- [ ] Clean architecture: infrastructure/ layer only

---

## Priority 2 — NEXT (after regime detection)

### 2a. ARS Rebuild on Clean Foundations
- [ ] Lock data splits permanently (never revisit):
  - IS:            Jan 2022 – Dec 2023
  - Validation OOS: Jan 2024 – Dec 2024
  - True Holdout:  Jan 2025 – present ← NEVER TOUCH until final sign-off
- [ ] Fix parameters by theory (not optimized):
  - RSI period = 14 (Wilder standard)
  - Aroon period = 25 (standard)
  - ATR period = 14 (standard)
  - Aroon thresholds = 70/30 (standard interpretation)
- [ ] Free parameters to optimize: SlCoef, RR only (≤ 2, ≤ 12 combinations)
- [ ] Fix objective function: Sharpe or Sortino (implement GetFitness() in cBot if needed)
- [ ] Fix spread assumption: use Binance-realistic 5 pips, NOT cTrader 490
- [ ] Fix commission: 0.04% per side (Binance futures taker rate)
- [ ] Fix commission: 0 is not acceptable — add to all future backtests
- [ ] Decide entry mode before optimizing: Breakout OR Pullback OR Combined (one at a time)
- [ ] Rerun OOS on Validation split only (not True Holdout)
- [ ] Permutation test (min 100 shuffles) before touching Holdout

### 2b. Multi-Timeframe ARS (MTF)
- [ ] BLOCKED until 2a is complete and passes permutation test
- [ ] Design: 4h direction · 1h signal · 15m entry
- [ ] Hypothesis doc required before any code

---

## Priority 3 — LATER

### 3a. Data Layer Module
- [ ] Binance OHLCV → Parquet (BTC/USDT, ETH/USDT, 15m/1h/4h)
- [ ] Funding rate (Binance API, free) — already in crypto data priority list
- [ ] Realized volatility (computed from OHLCV)
- [ ] Stablecoin supply growth (on-chain free APIs)
- [ ] Hurst exponent (computed from OHLCV — no new data source)

### 3b. Portfolio Layer
- [ ] Second strategy candidate (must be uncorrelated with ARS)
- [ ] Correlation matrix between strategy equity curves
- [ ] Portfolio-level risk monitor
- [ ] Target: 5–10 uncorrelated strategies (Renaissance lesson)

### 3c. Execution Layer
- [ ] Paper trading on Binance Futures
- [ ] Live execution on Binance Futures
- [ ] Streamlit UI enhancements
- [ ] OpenClaw signal integration

---

## Completed / Resolved This Session

- [x] ARS audit — full verdict delivered (see OVERFITTING_LESSONS.md)
- [x] Identified: 1,430 GA passes on 335B combination space = critical overfitting
- [x] Fixed: reduced to 2 free parameters (SlCoef, RR), 12 combinations
- [x] Fixed: chronological split corrected (IS=2022-24, OOS=2024-25)
- [x] Fixed: removed "Maximise Winning Trades" from objective
- [x] Quantified: spread destroys 68% of strategy profit on cTrader vs Binance
- [x] Decision confirmed: QuantStand on Binance is the correct execution path
- [x] CoinGlass research: liquidation heatmap > order book for swing trading reliability
- [x] CoinGlass added to high-priority data layer tasks

---

## Open Blockers (Still Unresolved)

| Blocker | Status |
|---------|--------|
| Hurst exponent on IS period (2022-2024 ETH h4) | Not yet computed — needs Python module |
| Sharpe on ARS OOS run | Not computable in cTrader — needs QuantStand module |
| CoinGlass API key | Not acquired — needed before data layer build |
| Regime classifier type (rule-based vs ML) | Decision deferred to regime design session |
| True Holdout date boundary formally locked | Defined in 2a task but not yet committed to repo |
