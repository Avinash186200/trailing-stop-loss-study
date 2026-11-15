# Trailing Stop-Loss Study

[![CI](https://github.com/Avinash186200/trailing-stop-loss-study/actions/workflows/ci.yml/badge.svg)](https://github.com/Avinash186200/trailing-stop-loss-study/actions/workflows/ci.yml) ![Python](https://img.shields.io/badge/python-3.11-blue) [![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)

I wanted to know whether a trailing stop helps a set of trades that already have their own entry and exit rules. This repo is the framework I built for that question: take each trade, replay the bars between entry and exit, and see where an ATR-based trailing stop would have taken it out instead.

Everything runs on synthetic data (a random-walk price path and random trades), so you can clone it and run it without any data files.

## What problem this solves

A trading strategy already decides when to enter and when to exit. A trailing stop is an extra exit that follows the price and closes the trade when it turns against you by some distance. The question is whether adding one would have made a set of trades better. This repo answers it by replaying each trade bar by bar between its entry and its original exit, and seeing where the stop would have closed it.

## What you will learn

- How an ATR-based trailing stop works: the initial stop, the trail, the cap, and the ratchet.
- Which timing rules matter in a stop backtest: when trailing starts, which bar's ATR you may use, and how a stop is filled when price gaps through it.
- How to resample 5-minute bars to 15-minute bars before running stop logic.
- How to compare trades with and without the stop (hit rate, mean, worst, sum of returns).
- How to search a parameter grid on an early sample, test once on a later sample, and read a sensitivity table.

## How it works

Read the notebooks in this order.

1. [`trailing_stop_backtest.ipynb`](notebooks/trailing_stop_backtest.ipynb): builds 250 days of synthetic 5-minute bars (a random walk with slowly changing volatility), resamples them to 15 minutes, computes a 14-bar ATR, and makes 120 random trades that enter at the last bar of a day and exit 2 to 9 trading days later. It then runs one parameter set (initial stop 1%, 3 x ATR trail, 3% cap) and compares the trades with and without the stop. Teaches: the stop logic itself.
2. [`trailing_stop_optimization.ipynb`](notebooks/trailing_stop_optimization.ipynb): 400 days of bars and 200 trades, split by entry time into 140 train and 60 test trades. It runs an 18-point grid (initial stop x ATR multiple x cap) on the train trades, picks the best mean return, tests that one setting on the test trades, and shows a sensitivity table. Teaches: honest parameter selection.

What the default seed shows, and why it is a lesson rather than a result:

- In the backtest notebook all 120 trades exit on the stop. The ATR comes from 15-minute bars, so 3 x ATR is a tight trail for a trade held for days. Hit rate falls from 55.8% to 41.7%, the worst trade improves from -6.4% to -1.0%, and the mean return falls from 0.196% to about 0.
- In the top ten rows of the grid, rows that differ only in `cap_pct` give identical results: for these trades the ATR trail is tighter than a 2% to 5% cap, so the cap never binds. A parameter that does nothing is worth spotting before you tune it.
- On the 60 test trades the stop again cuts the worst loss (-8.2% to -0.9%) and moves the mean to about 0.
- This is a random walk, so the stop should not add edge, and it does not. It changes the shape of the outcomes (smaller worst losses, fewer winners), not the expected value.

## Key concepts

- **Overlay:** a rule added on top of existing trades without changing their entries.
- **Initial stop:** a fixed distance from the entry price (`initial_pct`), active from the start.
- **ATR (average true range):** the average bar range, counting gaps from the previous close. A volatility measure in price units.
- **ATR trailing stop:** the stop sits `atr_mult x ATR` behind the best price since entry and only moves in the trade's favour (the ratchet).
- **Cap:** the trail is never looser than `cap_pct` from the best price.
- **Next-day trailing:** entries are at the end of the day, so the stop is only checked and moved from the next trading day.
- **Order of checks:** each bar is first checked against the stop set by earlier bars, and only then used to move the stop. A bar never moves a stop that it is itself tested against.
- **Gap-through fill:** if a bar opens beyond the stop, the fill is the open, not the stop level. Assuming the stop price would flatter the results.
- **Resampling to 15 minutes:** fewer, larger bars, so single 5-minute spikes trigger fewer stops.
- **In-sample / out-of-sample:** choose parameters on early trades, then test once on later trades.
- **Sensitivity table:** the train result around the chosen point. A plateau is more trustworthy than a single spike.

## Repo layout

```
notebooks/
  trailing_stop_backtest.ipynb       one parameter set, trade-level and summary comparison
  trailing_stop_optimization.ipynb   grid search on the early sample, one out-of-sample check, sensitivity table
tests/
  test_notebooks.py   every notebook is valid, output-free, with no credentials or local paths
docs/
  README.md           index of the docs
  methodology.md      the stop rules and the reasons behind each choice
original/             the original research notebooks (read-only record)
.github/workflows/ci.yml   runs the tests on every push
.env.example · requirements.txt · CHANGELOG.md · CONTRIBUTING.md · LICENSE
```

## How to run

```bash
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m pytest -q
jupyter lab notebooks/
```

Outputs go to `data/outputs/` at the repo root (git-ignored), whether you start Jupyter from the repo root or from `notebooks/`; set `DATA_DIR` to change that. To use your own data, replace the synthetic generators with a loader that returns the same columns (`open, high, low, close` bars and `trade_id, direction, entry_time, entry_price, exit_time, exit_price` trades). Notebooks are committed without outputs; run them to see the tables. The grid search takes a minute or two.

## What I learned

- Stop logic is full of small timing decisions (same-day trailing or not, gap fills, which bar's ATR you are allowed to use). Each one can move the result more than the parameters do.
- A grid search will always find a "best" setting. Picking it on the early sample and testing once on the later sample is the minimum; the sensitivity table tells you whether it is a plateau or a lucky spike.
- A trailing stop changes the shape of the return distribution (smaller worst losses, fewer big winners) more reliably than it changes the mean.

## Status

A research framework, finished for what it is. On random-walk data a stop should not add edge, and it doesn't; that is a useful sanity check of the code rather than a result.

## Original research code

[`original/`](original/README.md) holds the code as I first wrote it for this project. It is kept so you can
read the first version alongside the maintained one; some of it contains the issues in the audit table above,
and comparing the two shows each fix.

See [CONTRIBUTING.md](CONTRIBUTING.md) and the [CHANGELOG](CHANGELOG.md).
