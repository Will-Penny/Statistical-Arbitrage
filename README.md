# Statistical Arbitrage Pairs Trading — Pipeline & Falsification Test

A pairs-trading (statistical arbitrage) backtest across 50 large-cap US stocks, built around a
strict formation/trading split, and stress-tested with a placebo/permutation test to check whether
the edge is real or a screening artifact.

## Overview

Pairs are selected on a rolling **formation window** (correlation → same-sector filter → Engle-Granger
cointegration test → Ornstein-Uhlenbeck half-life filter) and then traded out-of-sample on the following
**trading window**, using z-score entries, a circuit breaker for structural breaks, and volatility-scaled
position sizing net of transaction costs. Individual pair PnL streams are combined into a portfolio via
walk-forward Markowitz optimization. The notebook documents the full process, including the look-ahead
bias mistakes that came up along the way and how they were fixed.

Key findings:

* The naive version of the strategy (no sector filter, no formation/trading split, no fees) showed no
  real edge once look-ahead bias and realistic costs were addressed.
* Adding a sector-match filter, a stricter cointegration threshold, and an OU half-life filter produced
  a modest net-of-cost edge: ~5.5% over 4 years unlevered, Sharpe ≈ 0.68.
* A placebo/permutation test — rerunning the entire pipeline 50 times with sector labels randomly
  shuffled — found the real strategy beat the shuffled-sector version in 48 of 50 trials (empirical
  p ≈ 0.04), suggesting the edge isn't purely a statistical artifact of the screening machinery.
* With only 50 placebo trials this is a coarse, borderline estimate, and two placebo draws actually beat
  the real result — the notebook is explicit that this is "suggestive, not confirmed."
* Known limitations that remain unresolved: survivorship bias (only currently-listed large caps were
  tested), possible over-tuning of filter thresholds to this specific historical window, and PnL
  discontinuity at the boundaries between formation/trading epochs.

## Data

Prices for 50 large-cap US tickers across 8 sectors (Tech, Communications, Consumer Discretionary,
Consumer Staples, Financials, Healthcare, Energy, Industrials) are pulled directly via `yfinance`
for 2018-01-01 to 2024-01-01. No static data files are needed — the notebook downloads everything
on run. Sector labels are hardcoded in a `SECTOR_MAP` dict at the top of the notebook and can be
edited to change the universe.

## Setup

```
pip install yfinance pandas numpy statsmodels scipy matplotlib tqdm
```

No API keys or manual downloads required — just run the notebook top to bottom (or run Part 8's
`run_pipeline` cell on its own if you only want the summary statistics, since it's self-contained).

## Structure

* **Part 1** — data acquisition (yfinance) and the sector map used for the economic prior filter
* **Part 2** — pair formation: correlation screen → same-sector filter → Engle-Granger cointegration
  test → OU half-life filter → diversification cap
* **Part 3** — turning a formed pair's spread into a signal: z-score entries, a ±4 circuit
  breaker/cooldown for structural breaks, volatility-scaled position sizing, and PnL net of
  transaction costs
* **Part 4** — a plain-English explanation of the hedge ratio underlying every pair's spread
* **Part 5** — portfolio construction: walk-forward Markowitz optimization across up to 10 active
  pairs at a time, including the covariance-matrix PSD regularization needed to make the optimizer
  numerically stable
* **Part 6** — a table of every look-ahead bias mistake that came up during development and how each
  was fixed; useful as a checklist for any backtest, not just this one
* **Part 7** — performance metrics (Sharpe, max drawdown, drawdown duration) and the equity curve
* **Part 8** — the placebo/permutation validation test: the entire pipeline (Parts 1–7) is
  encapsulated into a single `run_pipeline` function and rerun 50 times with the sector map randomly
  shuffled, to check whether the edge survives when the economic story is destroyed
* **Part 9** — honest summary of where the results landed, including what's still unresolved

## Limitations

* Universe is currently-listed large caps only — this is a real survivorship-bias problem, not just a
  caveat, since failed or delisted companies are excluded by construction.
* Filter thresholds (correlation > 0.90, ADF p < 0.01, half-life < 30 days, z-entry at 2.0) were chosen
  once and not cross-validated against a separate holdout period, so some of the edge could reflect
  fitting to this specific 2018–2024 window.
* Each formation/trading epoch is optimized independently; positions don't carry a memory of risk
  across epoch boundaries, which can create discontinuities in the portfolio's realized risk.
* The placebo test uses only 50 shuffled runs — enough to be suggestive (empirical p ≈ 0.04), not
  enough to be a rigorous significance test.
