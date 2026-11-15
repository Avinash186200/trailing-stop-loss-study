# Methodology

The stop rules and the reasons behind them. The README has the reading order and what the default run shows.

## Data

Synthetic and seeded (42). One instrument, 5-minute bars from 09:30 to 16:00 on business days, a random walk whose
volatility drifts slowly from day to day. There are no overnight gaps: each day opens at the previous close.
Trades enter at the close of the last 15-minute bar of a day (15:45), with a random direction, and the original
exit is the last bar of a day 2 to 9 trading days later.

## Bars and ATR

- 5-minute bars are resampled to 15 minutes (`label="left", closed="left"`; open first, high max, low min,
  close last). Fewer, larger bars mean fewer stop-outs from single spikes.
- ATR is the 14-bar simple mean of the true range: the largest of high - low, |high - previous close| and
  |low - previous close|.

## The stop, bar by bar

For each trade, take the 15-minute bars after entry up to and including the original exit bar, and drop any bar
on the entry day.

1. Start with the initial stop at `entry x (1 - direction x initial_pct)`.
2. For each bar, first check the current stop. A long is stopped if the bar's low reaches it, a short if the
   high does.
3. If stopped, the fill is the stop level, or the bar's open if the bar opened beyond the stop (gap-through).
4. If not stopped, update the best price (highest high for a long, lowest low for a short), then compute
   `trail = best - direction x atr_mult x ATR` and `cap = best x (1 - direction x cap_pct)`, take the tighter of
   the two, and move the stop only if that is in the trade's favour.
5. If no bar hits the stop, the trade keeps its original exit.

Checking before updating matters: a bar's own high and its own ATR only move the stop for later bars. Updating
first would let a bar's high raise the stop and then test that same bar's low against it, which is look-ahead
inside the bar.

## Metrics

Per trade: `direction x (exit / entry - 1)`, with and without the stop. Summary: number of trades, hit rate
(share above zero), mean, worst and sum of returns. No costs.

## Parameter search

- Trades are split by entry time: the first 70% are train, the rest test.
- Grid: `initial_pct` in {1%, 2%}, `atr_mult` in {2, 3, 4}, `cap_pct` in {2%, 3%, 5%}; 18 combinations.
- The best train mean return is picked and evaluated once on the test trades.
- The sensitivity table shows the train mean across `atr_mult` and `cap_pct` at the chosen `initial_pct`.

## Known limitations

- Random-walk data, so there is no edge to find; the run is a check of the code, not of the idea.
- No overnight gaps in the synthetic bars, so gap-through fills only happen when the stop moves above (or below)
  the next bar's open within a day. Real data gaps at the open far more often.
- ATR from 15-minute bars is small next to a multi-day hold, so on the defaults nearly every trade is stopped
  and the cap rarely binds. With real data you would try a daily ATR as well.
- The last few train trades can still be open when the test period starts, so a few train bars overlap the
  test period. With one parameter set chosen from a coarse grid the effect is small, but it is not zero.
- No costs or slippage.
