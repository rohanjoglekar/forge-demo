# Methodology — the rejection criteria are the product

This document explains how Forge decides that a strategy result can be
believed: the measured cost model, the data discipline, the multiple-testing
correction, and the negative results that stopped work. It exists because the
research pipeline has evaluated 33,000+ strategy variants over 540 days of
BTC data and almost all of them fail — that is the design, since a research
pipeline that mostly says yes is a random-number generator with extra steps.
Everything below exists to make a *yes* mean something.

## Trading costs are measured, not assumed

Every backtest charges the venue's actual fee formula — for Kalshi 15-minute
binaries, `ceil(0.07·C·P·(1−P))` per execution, verified against the official
fee schedule — plus a bid–ask half-spread. The half-spread is not a modeling
assumption; it comes from a collector that logs the live order book:

```python
# algo-trading/btc_lab/config.py
mid_half_spread_cents: float = 5.0    # against the trader, 0.10 <= p <= 0.90
tail_half_spread_cents: float = 4.25  # deci-cent tick region: p<0.10 or p>0.90
```

Those numbers used to be 0.5¢ and 0.1¢ — plausible-sounding, and wrong by an
order of magnitude. A study of the recorded book (n=398 tradable
entry-minute quotes; median full spread ~10–11¢) forced a recalibration to
5.0¢ and a
re-pricing of the entire leaderboard. The venue's top-ranked strategies went
from
**+$67.53 to −$26.50 overnight.** It shipped anyway, because the
alternative is a dashboard that lies at the exact moment a promotion
decision reads it. The best strategy's break-even spread (6.11¢) is now
displayed next to the measured one — that margin *is* the research question.

The original 0.1¢ "tail" number was a category error worth naming: below 10¢
the venue's tick size is 0.1¢, and the tick size had been written down as the
spread. The book can quote in fine ticks and still be 8¢ wide.

## Train / validation / holdout, and the holdout stays pristine

Data splits 60/20/20 chronologically. Search and refinement see only the
training segment; promotion tests read validation; holdout is spent only on
the final read of a candidate that has already passed everything else. Any
dashboard panel that compares exits or parameters is labeled with the segment
it reads, because a selection-driven number dressed up as out-of-sample is
the standard way quant research fools its author.

## 33,000 variants must pay for their own multiple testing

Selecting the best of N candidates clears any fixed significance bar by luck
alone as N grows, so the promotion test's t-statistic threshold rises with
the number of concurrent candidates:

```python
# algo-trading/btc_lab/pipeline.py
def _mt_floor(n_concurrent: int) -> float:
    """FWER-controlling promotion floor for N concurrent shadow tests: the
    one-sided Bonferroni tail quantile Φ⁻¹(1 − α/N), α=0.05. A pure-null
    best-of-N clears it only ~5% of the time — unlike √(2·ln N) (the EXPECTED
    max of N normals, a location statistic that ~half of null batches exceed).
    At N=1 it is 1.645 (a plain one-sided 5% test); it rises with N."""
```

The statistic under the threshold is a t-test on the mean P&L of each
strategy's **own realized payoffs**, net of modeled fees and spread. It
replaced a fixed hold-to-expiry break-even win rate that seven of ten live
strategies didn't even have (a stop-loss strategy's break-even is not 52.5%).
N — the concurrent test count — is fixed when a strategy enters shadow
testing (live predictions with no orders placed), so a threshold that rises
later cannot retroactively penalize a strategy that was already accumulating
evidence honestly.

## Results are stamped with the economics that produced them

When cost assumptions change, historical results stop being comparable. Each
venue carries an `ECONOMICS_VERSION`, every recorded result is stamped with
it, and every consumer — the leaderboard, best-performer selection,
re-evaluation, entry into shadow testing — treats a result from stale
economics as unusable rather than optimistically comparable. The engine
version alone can't do this job: engine changes that don't touch pricing
would falsely invalidate months of results, and pricing changes hidden in a
patch release would silently poison comparisons.

## Negative results are kept, and they stop work

Three studies exist specifically to *stop* things from being built:

- **Maker viability** — re-priced all 625 perpetual variants under
  best-case maker economics (fill-everything, 5 bps). 0 of 625 significant on
  holdout → the maker execution system that depended on this study does not
  get built.
- **Timeframe rescue** — swept 1h/2h/4h/6h bars over the perpetual strategy
  families, 870 evaluations, including a friction-halved control. 0 positive
  → the edge isn't hiding at another horizon.
- **BTC/ETH pair trading** — 202 spread-reversion configurations. Not viable.

The perpetual venue's verdict after 1,161 honest evaluations is currently
"fees ≈ loss, no surviving hypothesis except walk-forward retraining and
LLM-generated families, which are still competing through the normal gates."
The dashboard says this out loud. A research platform that can't display its
own dead ends will eventually trade one.

## Shadow testing is the same code path as backtesting

Strategies in shadow testing predict live 15-minute windows with no orders
placed,
through the same engine, same exits, same cost model as the backtest. A
reconciliation check flags any strategy whose live signal frequency diverges
from its backtest's, because that divergence is what look-ahead bias looks
like when it escapes into production.
