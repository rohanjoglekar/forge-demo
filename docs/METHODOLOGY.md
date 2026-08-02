# Methodology — the gates are the product

The lab has evaluated 33,000+ strategy variants over 540 days of BTC data.
Almost all of them die. That is the design: a research pipeline that mostly
says yes is a random-number generator with extra steps. Everything below
exists to make a *yes* mean something.

## Friction is measured, not assumed

Every backtest charges the venue's real fee formula — for Kalshi 15-minute
binaries, `ceil(0.07·C·P·(1−P))` per execution, verified against the official
fee schedule — plus a bid–ask half-spread. The half-spread is not a modeling
choice; it comes from a collector that logs the live order book:

```python
# algo-trading/btc_lab/config.py
mid_half_spread_cents: float = 5.0    # against the trader, 0.10 <= p <= 0.90
tail_half_spread_cents: float = 4.25  # deci-cent tick region: p<0.10 or p>0.90
```

Those numbers used to be 0.5¢ and 0.1¢ — plausible-sounding, and wrong by an
order of magnitude. When I studied the recorded book (n=398 tradable
entry-minute quotes; the median full spread is ~10–11¢), I recalibrated to
5.0¢ and re-priced the entire leaderboard. The book's champion set went from
**+$67.53 to −$26.50 overnight.** I shipped it anyway, because the alternative
is a dashboard that lies to me at the exact moment a promotion decision reads
it. The best variant's break-even spread (6.11¢) is now displayed next to the
measured one — the margin *is* the research question.

The original 0.1¢ "tail" number was a category error worth naming: below 10¢
the venue's tick size is 0.1¢, and I had written the tick size down as the
spread. The book can quote in fine ticks and still be 8¢ wide.

## Train / validation / holdout, and the holdout stays pristine

Data splits 60/20/20 chronologically. Search and refinement see train;
promotion gates read validation; holdout is spent only on the final read of a
candidate that already passed everything else. Any panel on the dashboard that
compares exits or parameters is labeled with which segment it reads, because a
selection-driven number dressed up as out-of-sample is the standard way quant
research fools its author.

## 33,000 variants must pay for their own multiple-testing

A best-of-N selection clears any fixed significance bar by luck alone as N
grows, so the promotion gate's t-statistic floor rises with the number of
concurrent candidates:

```python
# algo-trading/btc_lab/pipeline.py
def _mt_floor(n_concurrent: int) -> float:
    """FWER-controlling promotion floor for N concurrent shadow tests: the
    one-sided Bonferroni tail quantile Φ⁻¹(1 − α/N), α=0.05. A pure-null
    best-of-N clears it only ~5% of the time — unlike √(2·ln N) (the EXPECTED
    max of N normals, a location statistic that ~half of null batches exceed).
    At N=1 it is 1.645 (a plain one-sided 5% test); it rises with N."""
```

The statistic under the floor is a mean-P&L t-test on each variant's **own
realized payoffs**, net of modeled fees and spread. It replaced a fixed
hold-to-expiry break-even win rate that seven of ten live variants didn't even
have (a stop-loss variant's break-even is not 52.5%), and N — the concurrent
test count — is frozen at enrollment so a floor that rises later can't
retroactively tax a variant that was already accumulating evidence honestly.

## Results are stamped with the economics that produced them

When friction changes, every historical result is suddenly in a foreign
currency. Each venue carries an `ECONOMICS_VERSION`, every recorded result is
stamped with it, and every consumer — the leaderboard, champion selection,
re-evaluation, pipeline enrollment — treats a result from stale economics as
unusable rather than optimistically comparable. The engine version alone can't
do this job: engine changes that don't touch pricing would false-invalidate
months of results, and pricing changes hidden in a patch release would
silently poison comparisons.

## Negative results are kept, and they stop work

Three studies exist specifically so I would *stop* building things:

- **Maker viability** — re-priced all 625 perpetual variants under
  best-case maker economics (fill-everything, 5 bps). 0 of 625 significant on
  holdout → the maker execution stack the roadmap gated on this study does
  not get built.
- **Timeframe rescue** — swept 1h/2h/4h/6h bars over the perp families,
  870 evaluations, including a friction-halved control. 0 positive → the edge
  isn't hiding at another horizon.
- **BTC/ETH pair trading** — 202 spread-reversion configurations. Not viable.

The perpetual venue's verdict after 1,161 honest evaluations is currently
"fees ≈ loss, no surviving hypothesis except walk-forward retraining and
LLM-generated families, which are still competing through the normal gates."
The dashboard says this out loud. A research platform that can't display its
own dead ends will eventually trade one.

## Shadow testing is the same code path as backtesting

Enrolled variants predict live 15-minute windows with no orders, through the
same engine, same exits, same friction model as the backtest — and a
firing-rate reconciliation gate flags any variant whose live signal frequency
diverges from its backtest's, because that divergence is what look-ahead bias
looks like when it escapes into production.
