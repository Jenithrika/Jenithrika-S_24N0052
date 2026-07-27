# RegimeShift

This project tries to answer a simple question: can a portfolio do better by
figuring out what "mood" the market is in and adjusting itself accordingly,
instead of just sitting at a fixed 60/40 split forever?

The approach is to fit a Hidden Markov Model on NSE equity, gold, and a
liquid bond fund to infer three hidden regimes — Bull, Bear, and Crisis —
purely from price and volatility behavior, with no manual labeling. Each
regime gets its own mean-variance-optimized portfolio (solved with
`cvxpy`), and the whole thing is backtested walk-forward, so the model
never sees the future when deciding what "today" looks like. It's compared
against two simple benchmarks: a static 60/40 stock/bond split and an
equal-weight portfolio across all three assets.

Short version of the answer: it mostly doesn't beat them. That result —
and why — is the more interesting part of this project than the strategy
itself, and it's laid out honestly below rather than buried.

## Table of Contents

- [Results at a Glance](#results-at-a-glance)
- [A Few Decisions Worth Explaining](#a-few-decisions-worth-explaining)
- [Repository Structure](#repository-structure)
- [How to Run It](#how-to-run-it)
- [On Reproducing the Results Exactly](#on-reproducing-the-results-exactly)
- [How the Walk-Forward Validation Actually Works](#how-the-walk-forward-validation-actually-works)
- [Why the Dynamic Strategy Underperforms](#why-the-dynamic-strategy-underperforms)
- [Known Limitations](#known-limitations)
- [Further Exploration (Appendix)](#further-exploration-appendix--not-part-of-the-required-deliverables)

---

## Results at a Glance

| Strategy | Ann. Return | Sharpe | Sortino | Max Drawdown | Calmar | Ann. Turnover |
|---|---|---|---|---|---|---|
| Dynamic (regime, hysteresis) | 7.74% | 0.85 | 1.03 | -19.7% | 0.39 | 3.81x |
| Dynamic (regime, raw) | 6.51% | 0.76 | 0.95 | -16.8% | 0.39 | 4.33x |
| Static 60/40 | 8.52% | 0.89 | 1.12 | -22.9% | 0.37 | 0.06x |
| Equal Weight (1/3 each) | 9.49% | 1.29 | 1.62 | -14.4% | 0.66 | 0.08x |

*(All figures at 7.5bps transaction cost, 2013-04-30 to 2026-06-30, 3,216
trading days. The full table across all cost levels is in
`phase6_performance_summary.csv`.)*

The dynamic strategy doesn't beat either benchmark on risk-adjusted
return. It does hold up better than static 60/40 on drawdown and Calmar
ratio, and its best moment is the 2020 COVID crash, where it drew down
only -8.4% against static 60/40's -22.9%. But equal-weight — just
splitting evenly across the three assets and rebalancing quarterly —
beats everything on every metric across the full 13 years. More on why
below.

## A Few Decisions Worth Explaining

**Why three regimes instead of more?**
Bull, Bear, and Crisis map cleanly onto the project's core idea — calm,
declining, and acute stress each deserve a different portfolio. I didn't
push past three states mainly because of sample size: with roughly 3,200
usable trading days, splitting into more regimes starts leaving each one
with too few days to estimate a stable mean-variance solution from.

**Why these six features?**
I started with a bigger feature set and trimmed it down after checking
pairwise correlations — no point feeding the HMM two features that are
basically saying the same thing twice. What survived: `mom_21d` and
`mom_63d` for trend at two different horizons, `vol_21d` for how choppy
things have been recently, `drawdown_126d` to capture sustained decline
rather than just short-term noise, and `vix_level` plus `vix_chg_5d`
since the VIX carries forward-looking information that realized
volatility alone doesn't.

**Why `LIQUIDBEES.NS` as the "bond" leg, when it's really closer to cash?**
This one is a compromise I made on purpose, not by accident. I wanted a
real duration bond in the mix, but every gilt ETF I could find on Yahoo
Finance (`GILT5YBEES.NS`, `LTGILTBEES.NS`) only has 4–6 years of price
history. Using either of them would have quietly shrunk the whole
backtest down to an 18-month window with no real crisis in it — including
cutting out COVID entirely. I decided having the full 2010–2026 history,
crashes included, mattered more than having a "real" bond asset, so
LIQUIDBEES it is. It's a genuine limitation, and it's flagged again
further down.

**About the "Bear" label.**
When I looked at what the HMM was actually picking up for the Bear state,
its 63-day momentum is barely negative (+1.7%, if anything) and the
drawdown is mild (-5.6%). That's closer to "choppy, going nowhere in
particular" than an actual sustained decline. I left this in the notebook
as an explicit honesty check rather than quietly renaming the state to
something that sounds better than what it is — it's possible NSE's
history over this window just doesn't contain many genuine multi-year
bear markets that are distinct from short, sharp crisis episodes.

**Why hysteresis?**
The raw regime signal flips 104 times over the backtest, and a lot of
those are one- or two-day flickers right at a decision boundary — exactly
the kind of thing that costs more in trading fees than it's worth in
signal. Requiring a new label to hold for 3 consecutive days before the
portfolio actually acts on it brought confirmed transitions down to 86,
and it measurably helped: Sharpe went from 0.76 to 0.85, Sortino from
0.95 to 1.03.

## Repository Structure

```
RegimeShift.ipynb                       Primary deliverable: the full pipeline,
                                         Phases 1-6, top to bottom
regimeshift_appendix.py                 Further-exploration appendix (see below)
phase3_regime_overlay.png               HMM regimes over price (single global fit)
phase4_walkforward_regime_overlay.png   HMM regimes over price (causal, no lookahead)
phase6_equity_and_drawdown.png          Strategy vs. benchmark equity curves + drawdown
transition_matrix.csv                   HMM state transition probabilities
regime_labels.csv                       Per-day regime label (walk-forward)
phase6_performance_summary.csv          Full metrics table, all strategies x cost levels
PERFORMANCE_SUMMARY.md                  The required short performance summary
README.md                               This file
```

## How to Run It

This was built and run on **Google Colab**, so that's the easiest way to
reproduce it:

1. Open a new Colab notebook and upload `RegimeShift.ipynb`, or open it
   directly from GitHub via `File → Open notebook → GitHub`.
2. Run the first cell, which installs the packages Colab doesn't ship
   with by default:
   ```python
   !pip install yfinance hmmlearn cvxpy
   ```
3. Run all cells top to bottom (`Runtime → Run all`).

Phase 1 needs internet access to pull NSE, gold, VIX, and bond price data
via `yfinance` (2010-01-01 through 2026-06-30) — Colab has this by
default, no extra setup needed. Phase 4, the walk-forward HMM fitting, is
the slowest part of the notebook at roughly 100–110 seconds on a standard
Colab runtime — it's refitting a 3-state HMM with 10 random restarts
every 63 trading days across 13+ years of history, so that's expected.

If you'd rather run it locally instead of on Colab, the only difference
is you'll need `pip install yfinance hmmlearn cvxpy numpy pandas
matplotlib scipy` in your own environment first, then open the notebook
with Jupyter as usual.

## On Reproducing the Results Exactly

- The data is pulled fresh from Yahoo Finance every run. NSE/gold/VIX
  closing prices are historical and shouldn't change, so results should
  come out the same modulo any backfill or adjustment changes on Yahoo's
  end.
- Every HMM fit uses fixed `random_state` seeds (0 through however many
  restarts are configured), keeping the best-log-likelihood model — so
  given the same input data, the fit is deterministic.
- `cvxpy`'s solvers are deterministic for the convex QPs used here.

## How the Walk-Forward Validation Actually Works

This is the part of the project I was most careful about, since it's
easy to accidentally cheat here without noticing. The HMM is never fit
on data it's later evaluated against. Training starts at 750 days and
expands from there; every 63 trading days the model gets refit from
scratch on everything up to that point, and the feature scaler is refit
on that same training-only window — no full-sample statistics sneaking
in early. Each day's regime label is decoded causally, using only data
through that day, and then shifted forward one more day before the
portfolio is allowed to act on it. The notebook includes an explicit,
mechanical check confirming this: every training cutoff date is verified
to fall strictly before every date in its corresponding test window.

Transaction costs are applied on every rebalance as
`0.5 * sum(|target_weight - held_weight|) * cost_bps`, reported at 0, 5,
7.5, and 10bps. At 7.5bps, this costs the dynamic strategy roughly
40–50bps of annual return, driven mostly by its much higher turnover
(3.8–4.3x a year, versus under 0.1x for either benchmark).

## Why the Dynamic Strategy Underperforms

The short explanation: for over half the trading days in this window,
the strategy is sitting in Bear or Crisis and therefore holding a
comparatively defensive allocation — while NSE equity and gold were both
in strong, largely uncorrelated uptrends (correlation of -0.02) for most
of this period. That's exactly the environment where naive
diversification does great almost by accident, and where being cautious
costs you.

The strategy's real value shows up specifically in acute stress — the
COVID drawdown comparison above is the clearest example. Whether that
kind of protection is worth its cost in forgone return over a full
market cycle, and whether it's worth it compared to just holding a
simple diversified static portfolio, is really the central question this
project ends up answering. On this data, with this asset universe: not
quite. A more careful statistical look at that comparison — split by
era, with bootstrap confidence intervals — is in the "Further
exploration" section below.

## Known Limitations

- **The bond leg is basically cash, not a real duration bond**, for the
  data-availability reason explained above. This limits how much
  diversification the "bond" allocation can genuinely add during Bear or
  Crisis regimes.
- **"Bear" is probably better described as "choppy/consolidating"** given
  what the model actually found in NSE's history over this window.
  Treating it as defensively as Crisis was tried in the appendix and
  found to hurt more than help.
- **This is one historical path.** Every number here comes from a single
  13-year realization of Indian market history. The bootstrap analysis
  in "Further exploration" gives a sense of how much of the headline
  ranking is statistically real versus noise.
- **The regime-conditional portfolio weights are sensitive to the
  mean-return shrinkage parameter**, and that sensitivity isn't uniform
  across the walk-forward loop — a value that looks stable on the
  full-history fit doesn't necessarily hold for every individual refit
  window. Also explored in the appendix.

---

## Further Exploration (Appendix — Not Part of the Required Deliverables)

Everything in this section comes from `regimeshift_appendix.py`, a
separate script I used to dig further into the underperformance above
once the primary notebook was done. It's included because the process of
chasing this down was genuinely useful, but the primary notebook and its
results above stand on their own regardless of what follows.

### What I Tried

1. **Swapping in a real duration bond.** I tried replacing
   `LIQUIDBEES.NS` with an actual gilt ETF. Ran into the same
   data-history wall described above, so I reverted it.
2. **Lowering the Bear regime's risk aversion** (from 3.0 to 1.5), on the
   theory that if Bear is really "choppy" rather than declining, treating
   it that defensively didn't make sense.
3. **Fixing a rebalance-timing bug.** The original backtest engine only
   traded into an updated regime-to-weight table when the regime *label*
   changed — not when the table itself got refreshed on its regular
   re-fit schedule. That meant the portfolio could keep holding a stale
   allocation for weeks after the underlying weights had already moved.
   I fixed it to rebalance on either event.
4. **Sweeping the mean-return shrinkage parameter** from 0.1 to 0.5, to
   chase down a strange result where the Bear regime was briefly
   assigned more equity than Bull — the opposite of what should happen.

### What Actually Changed

| Strategy | Ann. Return | Sharpe | Sortino | Max Drawdown | Calmar | Ann. Turnover |
|---|---|---|---|---|---|---|
| Dynamic (hysteresis) — primary notebook | 7.74% | 0.85 | 1.03 | -19.7% | 0.39 | 3.81x |
| Dynamic (hysteresis) — appendix, final | 8.41% | 0.89 | 1.09 | -19.7% | 0.43 | 4.15x |

A modest improvement — Sharpe, Sortino, and Calmar all tick up a bit,
with drawdown basically unchanged and turnover slightly higher. The
rebalance-timing fix combined with hysteresis did most of the work here;
the shrinkage sweep, despite the effort, ended up round-tripping back to
the original value once I dug into why it wasn't actually solving the
problem I thought it was solving (more on that below).

### Splitting the Result by Era

| Period | Dynamic (hysteresis) Sharpe | Static 60/40 Sharpe | Equal Weight Sharpe |
|---|---|---|---|
| Pre-2020 (2013–2019) | 1.30 | 1.01 | 1.28 |
| COVID (2020) | 0.76 | 0.64 | 1.35 |
| Post-2020 (2021–2026) | 0.60 | 0.89 | 1.37 |

The strategy actually looks good pre-2020 and holds its own through
COVID — it's specifically the 2021–2026 stretch where it falls behind
both benchmarks. That lines up with the Bear-regime honesty check above
more than it points to some separate timing bug.

### How Confident Should We Be in Any of This?

| Strategy | Point Sharpe | 90% Confidence Interval |
|---|---|---|
| Dynamic (hysteresis) | 0.89 | [0.51, 1.41] |
| Static 60/40 | 0.89 | [0.36, 1.39] |
| Equal Weight | 1.29 | [0.87, 1.80] |

Using a block bootstrap (20-day blocks, 2000 resamples) to get a sense of
how much of this ranking is real: Dynamic-hysteresis and Static 60/40
turn out to be statistically indistinguishable, even after the
appendix's fixes. Equal Weight's interval sits above both, but it
overlaps them too — a real edge, but not one I'd want to overstate.

### Digging into Why Post-2020 Specifically Underperforms

I tested two theories directly instead of just guessing:

- **Theory 1 — the shrinkage parameter is too aggressive.** Sweeping
  `MU_SHRINKAGE` from 0.1 to 0.5 on the full-history checkpoint showed
  that the "correct" equity ordering (Bull holding more than Bear, Bear
  more than Crisis) only holds at shrinkage ≤ 0.3. Push it higher and
  all three regimes collapse toward "mostly cash," with Bear briefly
  overtaking Bull.
- **Theory 2 — the portfolio is just slow to react.** I tracked the
  actual held equity weight against the NIFTY price through the single
  worst post-2020 drawdown (peak on 2026-01-29, trough on 2026-03-23).
  Equity exposure sat at essentially 0% for the following ~90 trading
  days, right through a stretch where NIFTY recovered 5–7% off the low.

Here's the interesting part: re-running the backtest at the shrinkage
value the sweep recommended (0.3, which happens to be the notebook's
original setting) didn't meaningfully change the post-2020 numbers, or
that specific stuck-at-zero refit. What that tells me is the sweep's
check — which only looked at one fit on the entire 3,217-day history —
doesn't actually generalize to what a single walk-forward refit sees on
its own, much smaller training window. In this case, one refit trained
on a tiny 74/78/37-day Bull/Bear/Crisis split landed on a
corner-solution allocation of roughly 0% equity for Bear, and no single
global shrinkage value is going to reliably prevent that. I'm leaving
this as a flagged item for future work — something like a minimum
sample-size floor before trusting a given refit's weights, or shrinking
each new refit toward the previous refit's weights instead of toward the
cross-sectional mean — rather than something I tried to force-fix within
this project's scope.
