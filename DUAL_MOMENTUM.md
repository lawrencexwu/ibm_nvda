# Dual Momentum (GEM) — BOXX Defense · TradingView Indicator

A Pine Script v6 implementation of Gary Antonacci's **Global Equities Momentum**
(GEM) strategy, modified to hold **BOXX** (Alpha Architect 1–3 Month Box ETF, a
box-spread cash proxy) instead of AGG as the risk-off asset.

File: [`dual_momentum_gem_boxx.pine`](dual_momentum_gem_boxx.pine)

## Strategy rules

Evaluated once per **completed** month:

```text
US momentum    = SPY  12-month total return  (dividend adjusted)
Intl momentum  = VXUS 12-month total return  (dividend adjusted)
Cash hurdle    = BIL  12-month total return  (selectable)

If US momentum <= cash hurdle:
    Hold BOXX (defensive)
Else:
    Hold the stronger of SPY / VXUS
```

Execution convention: the signal is computed from the completed month-end
close, the switch happens at the next month's open, and the position is held
until the next rebalance. The synthetic portfolio return for any month is the
return of the asset selected at the *previous* month-end.

## What the indicator shows

- Current target allocation (SPY / VXUS / BOXX) and each asset's momentum
- Synthetic multi-asset equity curve vs. SPY buy-and-hold (TradingView's
  Strategy Tester can't rotate among multiple ETFs in one portfolio, so the
  backtest is computed inside the indicator)
- Stats table: CAGR, max drawdown, Sharpe, annualized volatility, monthly win
  rate, number of switches, months backtested, portfolio value
- Labels marking every allocation change, optional background coloring by
  allocation
- An alert that fires **only when the target asset changes** (both a static
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
| U.S. equity | `AMEX:SPY` |
| International equity | `NASDAQ:VXUS` |
| Defensive asset | `BATS:BOXX` |
| Cash hurdle | `BIL` (selectable: BIL / SHV / BOXX / FRED:DTB3 3-month T-bill yield / custom symbol) |
| Momentum lookback | 12 completed months |
| Rebalance frequency | every 1 month |
| Commission + slippage per switch | 0.05% + 0.05% |
| Starting portfolio value | 100,000 |
| Benchmark | `AMEX:SPY` |
| Display | stats table, table position, benchmark curve, switch markers, background coloring, momentum plots |

The cash hurdle decides whether equities are *eligible*; the defensive asset
is what is actually *held* during risk-off periods.

## Caveats

- **BOXX history is short** (inception Dec 2022). Every symbol needs
  `lookback + 1` completed months before the backtest starts, so with default
  settings the equity curve begins around Jan 2024. Set the defensive asset to
  `AGG` or `SHV` for a longer backtest window.
- The `FRED:DTB3` hurdle approximates the trailing 12-month T-bill return as
  the sum of monthly yield accruals (annualized yield ÷ 12), which is close to
  Antonacci's academic hurdle but not identical to an investable ETF return.
- Sharpe ratio is computed with a 0% risk-free rate from monthly returns.
