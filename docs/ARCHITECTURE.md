# Architecture — one box, honestly loaded

Everything runs on a single 4-core, 8 GB Linux VPS: the research daemon,
three trading runners, a FastAPI backend, two Next.js frontends, local LLMs
under Ollama, and the nginx edge. That constraint is not a limitation I plan
to apologize for — most of the interesting engineering below exists *because*
the box is small and shared.

## The service map

- `forge-btc-lab-daemon` — the governed research loop: search, backtest,
  shadow enrollment, promotion bookkeeping. Runs at `nice 15`, forever.
- `forge-knn-perp` + the binary shadow runner — live-market prediction on
  15-minute ticks; the money-adjacent path.
- `forge-dashboard-api` (:8700) / `forge-dashboard-web` (:3000) — FastAPI +
  Next.js, the private dashboard behind nginx Basic Auth.
- `forge-view` (:3001) — the public read-only demo, same components compiled
  with a view-only flag, behind a GET-only gateway.
- Ollama — local LLMs for the research committee and narrators, capped to one
  loaded model at a time, swap disabled (`MemorySwapMax=0`): on an 8 GB box a
  model that swaps is slower than no model.

## The governor that had to learn what "busy" means

The research daemon must yield to the live trader and the API, so it
originally throttled on the 1-minute load average. Measured over seven days
that was a disaster: load1 never fell below 4.0 and the daemon yielded 307
times in 5¼ hours — 73% of wall clock spent doing nothing. The bug is that
loadavg is **nice-blind**: ~74% of this box's CPU is nice-15 research, so the
daemon was yielding to *itself*.

The replacement reads `/proc/stat` and computes the fraction of capacity spent
on **non-nice** work — the live trader, the API, and Ollama's ~281% model-load
spikes — which is the only thing the daemon actually needs to get out of the
way of:

```python
# algo-trading/btc_lab/loop.py
def _nonnice_busy_frac(window_s: float) -> float | None:
    """Fraction of TOTAL cpu capacity spent on NON-NICE work over `window_s`.

    Why not the 1-minute load average, which this replaces: loadavg is
    nice-BLIND. On this box ~74% of CPU is `nice` — overwhelmingly the daemon's
    own nice-15 peers ... So `load1 > ncpu × frac` was a tautology: the daemon
    yielded because research was running, which is the thing it was trying
    to do."""
```

After the switch: zero spurious yields, eight strategy families explored in
the same wall clock that previously produced idle waiting. It falls back to
load1 if `/proc/stat` reads fail, because a governor that crashes governs
nothing.

## Deploys: push → Actions → one locked script

Every push to `main` runs a smoke-test workflow, then SSHes into the box and
runs `infrastructure/deploy.sh`: `git reset --hard`, re-assert the nginx
auth/routing invariants, restart services whose paths changed, rebuild the
demo. Two hardenings came out of one very bad day:

- **A lock.** The Actions deploy and a manual build once ran `npm` on the same
  tree concurrently and gutted it mid-reify. The script now takes `flock` on
  a lock file, and any manual script must take the same lock.
- **A postmortem worth retelling.** The demo kept dying with `next: not found`
  after every deploy for days, surviving any number of clean rebuilds. The
  root cause was a single tracked file: the demo's `node_modules` was a
  **committed symlink** to a path on my Mac. `.gitignore` had
  `node_modules/` — but a trailing-slash pattern matches only directories,
  not symlinks, so the link slipped into a commit, and every deploy's
  `git reset --hard` re-planted a dangling `/Users/...` path over whatever
  npm had just installed. One `git rm --cached` ended ten days of whack-a-mole.
  The lesson generalizes: when an artifact keeps vanishing on a box that runs
  `reset --hard`, ask git *what it thinks the path is* before blaming the
  package manager.

## The public demo is a build flag, not a fork

The demo shares the private dashboard's components. A compile-time
`VIEW_ONLY` flag strips mutating controls; the components are copied, not
branched, so every honesty fix lands in both. The landing page's live numbers
come from a purpose-built `/api/landing` endpoint that serves ~2 KB of counts
and thinned sparkline series, because the real section payloads (600 KB – 2.5
MB) are the wrong shape for a page whose job is a five-second first
impression — and a page that renders nothing until its fetch succeeds, so the
public landing cannot show a spinner or an error state.
