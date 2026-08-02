# Forge — a personal quant research & trading platform

**Live read-only demo: [forge-view.5.78.193.177.sslip.io](https://forge-view.5.78.193.177.sslip.io)** — real data, no login, nothing on it can place an order.

Forge is an automated trading-research platform, built and run by one person
on one Linux server. It generates trading-strategy variants, backtests them
against **measured** costs (real exchange fees plus the bid–ask spread
recorded from the live order book, not an estimate), shadow-tests the
survivors — live predictions with no orders placed — and promotes only the
strategies that clear statistical tests designed to punish luck. Three
automated paper-trading books then trade the output, 24/7, on real venues.

I built all of it: the research engines, the order-execution services, the
FastAPI backend, the Next.js dashboard, and the operations underneath. The
full source stays private because it runs live accounts; this repo is the
guided tour, with short excerpts quoted from the real code.

## The three books (live on the demo's front page)

| Book | Venue | What it does |
|---|---|---|
| **BTC 15-min predictor** | Kalshi binary markets | Trades the research pipeline's best-performing algorithm on 15-minute BTC up/down contracts. All stats reflect only the current strategy version, and are labeled that way. |
| **Equity trader** | Alpaca, $1M paper book | A long-only conviction book drawn from my 600-name research screen, with a dynamic T-bill reserve and an LLM reviewer in an advisory role. Scored against SPY, not against zero. |
| **Options trader** | Alpaca, cash-secured-put wheel | Scans, opens, manages and closes its own positions three times a day. Every decision is logged with its reason. |

All three trade paper money on real venues — the point is the machinery, and
the numbers are allowed to be red. The demo's landing page shows their live
P&L, including the flat one.

## What I'd want you to look at

- **[docs/METHODOLOGY.md](docs/METHODOLOGY.md)** — why most of this system
  exists to say *no*. The measured bid–ask spread that repriced my whole
  leaderboard from +$67 to −$26 (and why I shipped that number anyway), the
  significance threshold that rises with the number of strategies tested — so
  33,000 tries can't crown a winner on luck alone — and the three studies
  whose result was "don't build it."
- **[docs/SAFETY.md](docs/SAFETY.md)** — how a system that lets an LLM write
  and execute strategy code stays safe: a static code screen inside a
  bubblewrap sandbox with a read-only filesystem and no network access. Also
  the stop-loss ordering bug that produced four identical −$5,927 backtests,
  and the rule that enabling real execution is always a human act.
- **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** — everything on one 4-core,
  8 GB server: how the research daemon learned to yield CPU to the live trader
  (the 1-minute load average ignores process priority), and the deploy
  postmortem where a single committed symlink broke every deploy for ten days.
- **[docs/DESIGN.md](docs/DESIGN.md)** — the dashboard's honesty rules: every
  win rate carries its measurement window, a calibrated 4.25¢ never renders as
  4.3¢, and green is reserved for safe states — "LIVE" is not a color of
  success.

## What the demo deliberately doesn't show

The demo is the real dashboard compiled with a view-only flag — same
components, same live backend — minus everything that is my personal account
rather than the platform. What's excluded:

- **My Robinhood account.** The full dashboard's home is an account cockpit:
  real holdings with live P/L vs cost, a you-vs-S&P benchmark chart traced
  from actual deposits, a risk tab, orders and watchlists, and per-symbol
  position cards where the AI's verdict includes a recommended action for my
  own position (buy more / hold / trim) scored against my actual cost basis.
  The demo strips all of it — the equity screen shows model output only, and
  clicking a symbol shows research, never positions.
- **Real options positions.** The private Options view has three sub-tabs —
  my owned puts (as reported by the broker, per-contract P/L), the recommended
  cash-secured-put scan, and covered-call analysis on my real lots. The demo
  keeps only the scanner.
- **Live AI Q&A.** The private dashboard has free-form ask-anything panels
  and on-demand deep-dive regeneration (local LLMs + frontier models). Those
  require write requests, and the demo's gateway rejects all writes — so the
  demo serves cached analysis only, and says so where a button would be.
- **Controls.** Installing a proposed strategy, switching the active
  algorithm, enabling or disabling the trading services, tuning the promotion
  threshold — all compiled out of the demo build and rejected (HTTP 403) at
  the gateway anyway.
- **The perpetuals venue.** The research pipeline runs two venues; the demo
  pins to the 15-minute binaries because the perpetuals view exists mostly to
  document negative results (see METHODOLOGY) and needs that context to read
  fairly.
- **The rest of Forge.** The platform also runs a Discord agent fleet
  (research concierge, code agents), an MCP server for phone access, and a
  private investment-club system — none of which belong in a public surface.

## Stack

Python research engines (numpy/pandas, walk-forward and LLM-designed strategy
families) · FastAPI backend · Next.js 14 dashboard · systemd units and timers
on a single Linux VPS · local LLMs via Ollama, frontier models for design ·
GitHub Actions → self-locking deploy script.

## Also public

[voteconcordia](https://github.com/rohanjoglekar/voteconcordia) — a real-money
investment club decided by vote, executed in each member's own brokerage
account. Same author, same philosophy: fail open where it protects people,
fail closed where it protects money.

## Status

The research pipeline runs around the clock; the books trade paper on real
venues. Enabling live trading is a manual, human step and stays that way.

Contact: rohannj29@gmail.com
