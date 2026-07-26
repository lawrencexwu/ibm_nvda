# Dual Momentum (GEM) — QMOM/IMOM Holdings + BOXX Defense · TradingView Indicator

A Pine Script v6 implementation of Gary Antonacci's **Global Equities Momentum**
(GEM) strategy with two modifications:

1. **BOXX** (Alpha Architect 1–3 Month Box ETF, a box-spread cash proxy)
   replaces AGG as the risk-off holding.
2. **Signals and holdings are decoupled**: SPY / VXUS / BIL decide the regime
   and geography, while **QMOM / IMOM** (Alpha Architect quantitative momentum
   ETFs) provide the equity exposure actually held. This stacks cross-sectional
   stock momentum on top of GEM's time-series momentum.

File: [`dual_momentum_gem_boxx.pine`](dual_momentum_gem_boxx.pine)

## Strategy rules

Evaluated once per **completed** month, using dividend-adjusted total returns:

```text
US momentum    = SPY  12-month total return
Intl momentum  = VXUS 12-month total return
Cash hurdle    = BIL  12-month total return  (selectable)

If US momentum <= cash hurdle:        Hold BOXX
Else if US momentum >= Intl momentum: Hold QMOM
Else:                                 Hold IMOM
```

SPY vs BIL decides risk-on. SPY vs VXUS decides geography. QMOM or IMOM
provides the equity exposure. BOXX provides defense. Rebalance monthly.

## Modes

| Mode | Signals from | Portfolio holds | Purpose |
| --- | --- | --- | --- |
| **Baseline** | SPY / VXUS vs BIL | SPY / VXUS / BOXX | Classic GEM reference |
| **Decoupled** (default) | SPY / VXUS vs BIL | QMOM / IMOM / BOXX | Recommended: clean signals, factor holdings |
| **Aggressive** | QMOM / IMOM vs BIL | QMOM / IMOM / BOXX | Momentum measured directly on the holdings |

Backtest all three and compare — the mode is a single dropdown, so switching
is instant. The decoupled design deliberately keeps QMOM/IMOM out of the
regime decision: they are noisier and shorter-lived series than SPY/VXUS. The
trade-off is tracking divergence — the gate protects when *SPY* breaks down,
not when QMOM crashes relative to SPY (as momentum factor portfolios did in
2016 and 2023).

Execution convention: the signal is computed from the completed month-end
close, the switch happens at the next month's open, and the position is held
until the next rebalance. The synthetic portfolio return for any month is the
return of the asset selected at the *previous* month-end.

## What the indicator shows

- Current target holding (QMOM / IMOM / BOXX) and the momentum of each series
  in use, including the cash hurdle
- Synthetic multi-asset equity curve vs. SPY buy-and-hold (TradingView's
  Strategy Tester can't rotate among multiple ETFs in one portfolio, so the
  backtest is computed inside the indicator)
- Stats table: mode, CAGR, max drawdown, Sharpe, annualized volatility,
  monthly win rate, number of switches, months backtested, portfolio value
- Labels marking every allocation change, optional background coloring by
  allocation
- An alert that fires **only when the target ETF changes** (both a static
  `alertcondition` and a dynamic `alert()` naming the new asset)

## How to install

1. In TradingView, open **Pine Editor**, paste the contents of
   `dual_momentum_gem_boxx.pine`, and click **Add to chart**.
2. Use a **monthly (1M)** chart of SPY. Daily/weekly charts also work; the
   table warns if the chart timeframe is above monthly.
3. To get alerts, create an alert on the indicator with the condition
   **"GEM: allocation change"** (or "Any alert() function call"), frequency
   *Once per bar*.

## Non-repainting design

- All symbol data is requested on the monthly timeframe as `close[1]` with
  `lookahead = barmerge.lookahead_on` — the standard confirmed-bar pattern
  that returns only the **last completed month**, never the partial one.
- All prices are dividend adjusted via
  `ticker.modify(symbol, adjustment = adjustment.dividends)` so momentum
  compares total returns, which matters for BIL, BOXX, and VXUS.

## Inputs

| Input | Default |
| --- | --- |
| U.S. signal | `AMEX:SPY` |
| International signal | `NASDAQ:VXUS` |
| U.S. holding | `BATS:QMOM` |
| International holding | `BATS:IMOM` |
| Defensive holding | `BATS:BOXX` |
| Cash hurdle | `BIL` (selectable: BIL / SHV / BOXX / FRED:DTB3 3-month T-bill yield / custom symbol) |
| Mode | Decoupled |
| Momentum lookback | 12 completed months |
| Rebalance frequency | every 1 month |
| Commission + slippage per switch | 0.05% + 0.05% |
| Starting portfolio value | 100,000 |
| Benchmark | `AMEX:SPY` |
| Display | stats table, table position, benchmark curve, switch markers, background coloring, momentum plots |

The cash hurdle decides whether equities are *eligible*; the defensive
holding is what is actually *held* during risk-off periods.

## Caveats

- **BOXX history is short** (inception Dec 2022) and **QMOM/IMOM launched
  Dec 2015**. Every symbol in use needs `lookback + 1` completed months
  before the backtest starts, so with default settings the equity curve
  begins around Jan 2024. Set the defensive holding to `AGG` or `SHV` for a
  longer window — and even then, the QMOM/IMOM live history covers barely one
  factor cycle, so do not overtrust the backtest.
- The absolute-momentum gate only tests the **U.S. signal** against the
  hurdle (Antonacci's original design): the model can hold IMOM while it
  falls, as long as SPY's trailing year still beats cash.
- The `FRED:DTB3` hurdle approximates the trailing 12-month T-bill return as
  the sum of monthly yield accruals (annualized yield ÷ 12), which is close to
  Antonacci's academic hurdle but not identical to an investable ETF return.
- Sharpe ratio is computed with a 0% risk-free rate from monthly returns.
