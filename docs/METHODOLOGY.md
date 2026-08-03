# Methodology — statistical validity before strategy promotion

This methodology document defines how Forge determines whether a quantitative
strategy result is statistically credible and eligible to advance from
research into shadow testing and automated paper trading. It covers measured
transaction costs, chronological data isolation, family-wise multiple-testing
correction, versioned market economics, and the retention of negative findings.
The pipeline has evaluated more than 33,000 strategy variants across 540 days
of BTC market data, and the overwhelming majority fail. That rejection rate is
intentional; promotion is reserved for results that remain defensible after
costs, selection effects, and live out-of-sample observation.

## Trading costs are measured, not assumed

Every backtest applies the venue's actual fee schedule. For Kalshi 15-minute
binary contracts, that is `ceil(0.07·C·P·(1−P))` per execution, independently
verified against the published schedule, plus a bid–ask half-spread measured
by a collector that records the live order book:

```python
# algo-trading/btc_lab/config.py
mid_half_spread_cents: float = 5.0    # against the trader, 0.10 <= p <= 0.90
tail_half_spread_cents: float = 4.25  # deci-cent tick region: p<0.10 or p>0.90
```

The original values were 0.5¢ and 0.1¢: superficially plausible assumptions
that understated observed execution costs by approximately one order of
magnitude. Analysis of 398 tradable entry-minute quotes found a median full
spread of approximately 10–11¢, requiring recalibration to 5.0¢ and complete
repricing of the leaderboard. The venue's highest-ranked strategy moved from
**+$67.53 to −$26.50** under the corrected economics. The result was published
unchanged. A promotion system is only useful if adverse recalibrations are
reflected immediately rather than suppressed. The dashboard now displays the
leading strategy's 6.11¢ break-even spread beside the measured spread because
the difference between those values defines the remaining execution margin.

The original 0.1¢ tail value also exposed a category error: the venue's tick
size below 10¢ had been treated as its spread. Fine quote increments do not
imply narrow liquidity; an order book that moves in 0.1¢ ticks can still be
8¢ wide.

## Chronological train, validation, and holdout isolation

Data is divided chronologically into 60% training, 20% validation, and 20%
holdout segments. Search and refinement are restricted to training data;
promotion tests operate on validation data; and holdout data is consumed only
for the final assessment of a candidate that has already cleared every prior
gate. Dashboard comparisons identify the segment they use, preventing
selection-conditioned results from being presented as out-of-sample evidence.

## Multiple-testing control across 33,000 variants

As the number of candidates increases, selecting the best result will
eventually clear any fixed significance threshold through chance alone.
Forge therefore raises the promotion t-statistic floor with the number of
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

The underlying statistic is a t-test on the mean P&L of each strategy's
**realized payoff distribution**, net of fees and measured spread. This
replaced a fixed hold-to-expiry break-even win rate that was structurally
inapplicable to seven of ten live strategies; a strategy with protective exits
does not share the 52.5% break-even rate of a hold-to-expiry strategy. The
concurrent test count, N, is fixed when a strategy enters shadow testing so
that subsequent growth in the candidate pool cannot retroactively alter the
evidentiary threshold under which observations were collected.

## Results are stamped with the economics that produced them

When execution economics change, historical results cease to be directly
comparable. Each venue defines an `ECONOMICS_VERSION`, every result is stamped
with that version, and all downstream consumers — leaderboard ranking,
candidate selection, reevaluation, and admission to shadow testing — reject
results produced under stale economics. Engine versioning is intentionally
separate: non-economic engine changes should not invalidate months of pricing
history, while a pricing correction must never remain hidden inside an
otherwise compatible release.

## Negative findings terminate unsupported work

Three completed studies produced explicit no-go decisions:

- **Maker viability** — Repriced all 625 perpetual variants under favorable
  maker assumptions: complete fills at 5 basis points. None achieved holdout
  significance, so the dependent maker-execution system was not built.
- **Timeframe sensitivity** — Evaluated the perpetual strategy families across
  1h, 2h, 4h, and 6h bars in 870 experiments, including a half-friction
  control. None produced a positive result, providing no evidence that the
  proposed edge existed at another horizon.
- **BTC/ETH pair trading** — Tested 202 spread-reversion configurations; none
  met the viability threshold.

After 1,161 evaluations, the current perpetual-futures conclusion is that
losses approximately equal fees, with no surviving hypothesis beyond
walk-forward retraining and LLM-generated families that remain subject to the
standard gates. The dashboard presents that conclusion directly. Preserving
negative findings prevents rejected hypotheses from quietly re-entering the
execution roadmap.

## Shadow testing is the same code path as backtesting

Strategies under shadow testing generate predictions for live 15-minute
windows without placing orders. They use the same engine, exit logic, and cost
model as their backtests. A reconciliation process flags material divergence
between live and historical signal frequencies, providing an operational
control for detecting look-ahead bias or implementation drift after a model
enters production data flow.
