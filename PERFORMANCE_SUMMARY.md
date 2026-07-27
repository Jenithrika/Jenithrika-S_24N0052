# Performance Summary

Backtest window: 2013-04-30 to 2026-06-30 (3,216 trading days), walk-forward
regime detection (no lookahead), transaction costs applied per rebalance.

## Headline comparison (7.5bps transaction cost)

| Strategy | Ann. Return | Ann. Vol | Sharpe | Sortino | Max Drawdown | Calmar | Ann. Turnover |
|---|---|---|---|---|---|---|---|
| **Dynamic (regime, hysteresis)** | 7.74% | 9.10% | 0.85 | 1.03 | -19.7% | 0.39 | 3.81x |
| Dynamic (regime, raw) | 6.51% | 8.55% | 0.76 | 0.95 | -16.8% | 0.39 | 4.33x |
| Static 60/40 | 8.52% | 9.60% | 0.89 | 1.12 | -22.9% | 0.37 | 0.06x |
| Equal Weight (1/3 equity, gold, bond) | 9.49% | 7.37% | 1.29 | 1.62 | -14.4% | 0.66 | 0.08x |

## Sensitivity to transaction cost

| Strategy | 0bps Sharpe | 5bps Sharpe | 7.5bps Sharpe | 10bps Sharpe |
|---|---|---|---|---|
| Dynamic (hysteresis) | 0.88 | 0.86 | 0.85 | 0.84 |
| Dynamic (raw) | 0.80 | 0.77 | 0.76 | 0.75 |
| Static 60/40 | 0.89 | 0.89 | 0.89 | 0.89 |
| Equal Weight | 1.29 | 1.29 | 1.29 | 1.29 |

The dynamic strategies are meaningfully more cost-sensitive than either
benchmark, a direct consequence of 40-70x higher annual turnover.

## Bottom line

The dynamic regime-switching strategy does not beat either fixed-weight
benchmark on risk-adjusted return over the full sample. It does hold a
shallower drawdown than static 60/40 (-19.7% vs -22.9%) and a better Calmar
ratio (0.39 vs 0.37), and its advantage is clearest during the 2020 COVID
crash specifically, where it drew down only -8.4% against static 60/40's
-22.9%. It underperforms equal-weight on every metric across the full
sample. See `README.md` for the regime-detection diagnostics, methodology,
and discussion of why.
