# Safety — letting an LLM write code without letting it drive

Forge has two kinds of dangerous surface: strategy code written by a language
model and executed automatically, and runners that could, if armed, spend real
money. The rules for both are mechanical, not aspirational.

## The codegen sandbox

One research dimension has a local LLM write novel `strategy(df, **params)`
bodies — free-form Python, evaluated like any other candidate. Executing
model-written code is the sharpest edge in the system, so it runs behind
layers that assume the body is adversarial:

1. **AST screen** — imports and attribute access are checked against an
   allowlist/blocklist before anything runs. No `os`, no `subprocess`, no
   dunder tunneling.
2. **Process jail** — the body runs in an isolated subprocess with rlimits
   (CPU seconds, address space, file size, descriptor and process counts) and
   `nice 15`.
3. **A dumb result channel** — the child writes arrays to a file; the parent
   reads them with `np.load(allow_pickle=False)`. No pickles, no objects, no
   deserialization gadgets.
4. **OS isolation** — on the VPS the subprocess runs under bubblewrap:

```python
# algo-trading/btc_lab/codegen.py
return [bwrap, "--ro-bind", "/", "/", "--dev", "/dev", "--proc", "/proc",
        "--bind", workdir, workdir, "--chdir", workdir,
        "--unshare-net", "--unshare-pid", "--die-with-parent", "--"]
```

Read-only filesystem: a body that somehow defeats the AST screen still cannot
modify trader state or config. No network: whatever it reads, it has nowhere
to send. Only its scratch workdir is writable, because the result file has to
live somewhere. The honest admission in the code's own comment: a pure-Python
screen cannot bound the transitive attribute surface of an adversarial body —
which is exactly why the OS layer exists and the screen is merely the first
tripwire.

## Arming is always a human act

No code path promotes itself into placing orders. The research pipeline's
terminal state is *proposed*; a human installs a proposed variant, and a
human arms a runner. The live-execution configs sit behind independent
switches (an execution-mode flag and a runtime armed flag), and the public
demo build strips every mutating control at compile time. This is the same
rule as my other project's money path: fail open where it protects
availability, fail closed where it protects money.

## The exit-stack integrity bug, kept as a scar

An audit found that on funding-boundary bars, the perpetual backtest engine
flattened positions *before* evaluating protective exits — so stop-losses were
silently skipped exactly where volatility clusters. Four different exit
configurations produced literally identical −$5,927.27 backtests, which is the
tell: their stops had never fired. Worse, two unit tests had been written
against the buggy behavior and were locking it in.

The fix reordered the bar loop — protective exits run first, then the funding
flatten — mirrored identically in the backtest engine and the live runner, and
added a gap-through rule (a trailing stop gapped past fills at the open, not
at the stop price, because you don't get your stop price in a gap). The
regression tests now encode the *correct* ordering, and the incident is why
"backtest and shadow share one code path" is a safety property here, not a
convenience.

## The public demo can't hurt anything, by construction

The demo you can click around is defended in three independent layers, because
any one of them failing should not be interesting:

1. **Gateway** — nginx accepts GET/HEAD only (mutations get 403 before
   reaching the application), 404s the audit endpoint, and rate-limits per IP.
2. **Build** — the demo frontend compiles with a view-only flag that removes
   arm/install/regenerate controls from the bundle rather than hiding them.
3. **Data** — the pages it renders are research output and paper-account
   performance; account surfaces (holdings, positions, cost basis, "your
   book" advice) are stripped server- and client-side.

TLS via Let's Encrypt; the bare hostname redirects to the demo instead of
falling through to the real dashboard's auth prompt — a visitor should never
meet a password dialog they weren't invited to.
