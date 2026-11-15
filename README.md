# Trailing Stop-Loss Study

[![CI](https://github.com/Avinash186200/trailing-stop-loss-study/actions/workflows/ci.yml/badge.svg)](https://github.com/Avinash186200/trailing-stop-loss-study/actions/workflows/ci.yml) ![Python](https://img.shields.io/badge/python-3.11-blue) [![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)

I wanted to know whether a trailing stop helps a set of trades that already have their own entry and exit rules. This repo is the framework I built for that question: take each trade, replay the bars between entry and exit, and see where an ATR-based trailing stop would have taken it out instead.

Everything runs on synthetic data (a random-walk price path and random trades), so you can clone it and run it without any data files.

## How the overlay works

- Entries happen at the end of the day. The initial stop is placed at a fixed percentage from the entry price.
- Trailing starts on the next trading day, never on the entry day.
- The trail sits `atr_mult x ATR` behind the best price since entry, capped so it is never looser than `cap_pct` from that best price, and it only ratchets in the trade's favour.
- If a bar gaps through the stop, the fill is the bar's open, not the stop level.
- If the stop is never hit, the trade keeps its original exit.
- 5-minute bars are resampled to 15 minutes first, which cuts noise-driven stop-outs.

## Structure

```
notebooks/
  trailing_stop_backtest.ipynb       one parameter set, trade-level and summary comparison
  trailing_stop_optimization.ipynb   grid search on the early sample, one out-of-sample check, sensitivity table
```

## How to run

```bash
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
jupyter lab notebooks/
```

Outputs go to `data/outputs/` (git-ignored); set `DATA_DIR` to change that. To use your own data, replace the synthetic generators with a loader that returns the same columns (`open, high, low, close` bars and `trade_id, direction, entry_time, entry_price, exit_time, exit_price` trades).

## What I learned

- Stop logic is full of small timing decisions (same-day trailing or not, gap fills, which bar's ATR you are allowed to use). Each one can move the result more than the parameters do.
- A grid search will always find a "best" setting. Picking it on the early sample and testing once on the later sample is the minimum; the sensitivity table tells you whether it is a plateau or a lucky spike.
- A trailing stop changes the shape of the return distribution (smaller worst losses, fewer big winners) more reliably than it changes the mean.

## Status

A research framework, finished for what it is. On random-walk data a stop should not add edge, and it doesn't; that is a useful sanity check of the code rather than a result.

## Original research code

[`original/`](original/README.md) holds the code as I first wrote it for this work, sanitised and nothing else:
client, company and person names replaced, credentials and local paths removed, data files and notebook outputs
dropped. It is kept so the real work can be read alongside the maintained version. Some of it contains the issues
described above; comparing the two shows each fix.

See [CONTRIBUTING.md](CONTRIBUTING.md) and the [CHANGELOG](CHANGELOG.md).
