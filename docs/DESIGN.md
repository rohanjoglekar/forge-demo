# Design — a dashboard that isn't allowed to flatter its owner

This document sets out the dashboard's design rules and the incidents that
produced them. Promotion and go-live decisions get made off this dashboard,
so its design rules are really research rules. The ones defended hardest:

## Every number carries its window

A win rate with no stated time window changes meaning every time the counters
reset.
Forge's measurement start date is explicit: the Kalshi book's stats are
measured from the day the current strategy version went live, the backend
sends that start date with the data, and every figure that uses it says
*since Jul 11* in its label. The account's lifetime spans two strategy
versions — 51% of its orders predate the current one — so an unlabeled
lifetime number would be arithmetic on apples and oranges.

The same rule caught a real bug in the public demo: a stale component rendered
**"P/L vs $1,000: +$191.11" next to $967 of equity** — two true numbers whose
juxtaposition was a lie (the +$191 covered only the current strategy version;
the $1,000 was the lifetime starting stake). The fix labels the P/L against
the equity held *on the measurement start date* ($776.21), so the three
numbers on that card now add up. Anyone should be able to do the arithmetic
on any card and have it close.

## One vocabulary for trading modes, and green means safe

Early on, three tabs described the same situation three ways: "LIVE · Alpaca
paper" in green, "PAPER · EXECUTING" in red, "live · disarmed" in lowercase.
The badge system now has one grammar — money reality (`LIVE / PAPER / SHADOW`)
times behavior (`EXECUTING / ADVISORY / DISARMED`) — and one color rule:
**color encodes risk, not excitement.** Real money executing is red, always;
green is reserved for states that cannot spend. "LIVE" as a data-source brag,
rendered in the color other tabs use for safety, was the exact confusion a
tired owner doesn't need at midnight.

## Calibrated parameters render exactly

The measured tail half-spread is 4.25¢. A formatter that rounds it to "4.3¢"
turns a calibrated constant into an apparent config mismatch for anyone
auditing the model — so the cents formatter keeps real decimals and drops only
trailing zeros. The same class of rule: raw floats never leak
(`0.4200000000000001` appeared in the order-book table once; prices are
formatted, period), and a strategy's name may be shortened in a summary card
only when the full configuration is one click away.

## Logs show signal, not ceremony

The shadow-test log (live predictions with no orders placed) records every
prediction, including one strategy — a vote across several models — that
records PASS, meaning no position, every window. Eighty percent of the raw
log was that strategy saying "nothing to do." The public demo defaults to
*trades only* with a one-click toggle to the full log; the private working
dashboard
defaults to everything, because PASS frequency is monitoring signal for the
operator and noise for a first-time reader. Same data, different defaults, both
labeled with what's hidden.

## Tables must fit the screen they're read on

The shadow-test table's win-rate, z and P&L columns — the columns the tab
exists for — were 300px off-screen at laptop width, with no visual hint to
scroll. Dropping a redundant column, wrapping the strategy names, and a
denser padding mode brought an 11-column table from 1,699px to fitting in
1,069px. A dashboard that hides its own conclusions off-screen is
indistinguishable from one that doesn't have any.

## The landing page cannot break

The public demo's first screen renders its prose with zero fetches; the live
elements (three books' P&L, per-card stat chips) render *nothing* until their
data arrives. No spinners, no error cards, no layout shift. A recruiter gives
the page five seconds; none of them should go to spinners or error states.
