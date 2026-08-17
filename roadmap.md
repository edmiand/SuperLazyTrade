# Roadmap: Win Rate Improvement

## Context

Win rate dropped to ~34% BUY / 38% SELL after the strict-alternation change
(`1d4fa3e`, `2a26b21`, `9ae6fd2`). Reviewed a 2026-08-15 SOXX 2-min chart
(EMA Cross anchor) showing two distinct failure modes:

1. **Open chop (09:30–10:30):** three whipsaw signals (SELL, BUY, SELL),
   all losers, during a wide-ranging, low-trend-strength period.
2. **Missed trend (11:30–16:00):** a long, clean, low-volatility uptrend
   (~1%) went almost entirely untraded. Only one BUY fired, at 15:00, with
   a WEAK score (26), capturing just 0.14%. Cause: strict alternation
   (CLAUDE.md "Signal blocking") requires the opposite signal to actually
   fire before a same-direction re-entry is allowed — if SELL's
   score/quality filter never clears during a strong one-directional move,
   BUY stays locked out for the whole trend.

Net effect: the system is willing to trade in chop but gets stuck sitting
out real trends — dragging win rate down from both directions at once.

## Candidate Ideas (not yet implemented)

Ranked roughly by leverage-to-effort.

### 1. Regime gate before signal generation (highest leverage)
Don't just penalize weak-regime scores — hard-block new entries when
ADX/Squeeze indicate chop (e.g. ADX below threshold AND price inside a
recent range). Currently these only subtract points via the gate system;
a hard "no trade" state would directly kill the open-chop losses seen at
09:30–10:30.

### 2. Opening-range exclusion
Exclude the first 15–30 minutes of RTH from new entries (or require a
confirmed opening-range breakout first). The entire chop-loss cluster on
the reviewed chart sits inside this window. Pure session-filter change,
no scoring logic touched.

### 3. Trend continuation / pyramiding (biggest architectural change)
Strict single-position alternation means the system is binary: right at
the flip or wrong all day. Consider allowing re-entry on pullback-in-trend
without requiring a full opposite signal to fire first — this is what
would have captured the 11:30–16:00 move. Bigger redesign than #1/#2;
touches the alternation/blocking logic described in CLAUDE.md.

### 4. Volatility-adjusted / trailing exits instead of fixed % target
Fixed `pnlTarget` rarely hits in a low-ATR grind session, so trades often
get closed flat/small-positive by the next opposite signal and get bucketed
as a loss under the current win-rate rule (anything without a PROFIT hit
counts as loss). An ATR-scaled target or trail-once-in-profit stop could
convert some of these into recorded wins.

### 5. Look at expectancy, not just win rate
Before optimizing further, get average win size vs average loss size —
win rate alone can mislead (e.g. tightening filters could shrink both
losing AND winning trade count/size). Worth instrumenting before picking
a fix.

## Applied (2026-08-15)

Two default-value changes, no new logic — the fastest lever available
using code that already existed:

- **`gateOn` default `false` → `true`** (Enable Risk Gates). Gate 1
  (Squeeze, −30) and Gate 4 (Liquidity, −25) directly target the
  09:30–10:30 chop regime from the reviewed chart — low-ADX,
  range-bound conditions now get penalized out of the score before a
  signal can fire, instead of only showing as a warning.
- **`emaSlowPeriod` default `20` → `30`** (EMA Slow Period). The
  tooltip's own documented comparison (fewer signals, ~40% less
  whipsaw risk, better SELL win rates) targets the same whipsaw
  cluster directly.

Neither change touches the strict-alternation lockout behind the
missed-trend failure mode (11:30–16:00) — that requires #1 (hard
regime gate before signal generation) or #3 (pyramiding/re-entry
redesign) below, which are architecture changes, not default tweaks.
Re-baseline win rate against these two new defaults before deciding
whether further work is still needed.

## Per-Asset Adaptivity (2026-08-16 exploration)

Separate line of investigation, prompted by reviewing the full tracking
watchlist rather than a single chart. AUTO mode currently adapts *only*
the SuperTrend ATR length/factor, and does so by matching ticker
strings. Everything else — VWAP limits, ADX floor, RVOL gate, stretch
scale, fuel cap, component weights, `pnlTarget` — is a static lookup on
`syminfo.type`, i.e. five fixed buckets.

Watchlist (traded: 14 stocks + 4 futures + SOXX; context-only:
SPCFD:SPX, TVC:NDQ, TSX:TSX).

Findings:

- **The AUTO SuperTrend table misses most of the list.** AMD, MU, META,
  HOOD, PLTR, IBM, SHOP and SPCX all fall through to the generic stock
  `else` at `10 / 2.5` — so SPCX gets the same SuperTrend as IBM. Note
  the generic default is *identical* to the AAPL/MSFT/AMZN/GOOGL branch,
  so for stocks the table has only two distinct outcomes in practice:
  NVDA/TSLA at `8 / 2.2`, and everything else at `10 / 2.5`. Not a
  defect — the `else` branches are deliberate — but it means the
  per-ticker tuning is thinner than the table's length suggests. All
  four traded futures (CL1!, NG1!, GC1!, SI1!) *are* matched correctly
  by the 2-char prefix logic.
- **One STOCK profile spans a ~5x volatility range.** SPCX and IBM share
  `vwap_h_limit = 1.8%` and `ext_scale = 3.0`. For IBM 1.8% from VWAP is
  rare, so Component 2 is effectively always full points; for SPCX it is
  routine, so it is effectively always reversion risk. Same problem in
  FUTURES, where NG1! and GC1! share `vwap_h_limit = 1.5%` despite NG
  having roughly 3–4x the daily range.
- **TVC:NDQ and TSX:TSX carry no volume** (SPCFD:SPX does — confirmed
  against live feeds). This is already handled, though only by accident:
  `rel_vol_rolling` (line 549) falls through its `vol_ma20 > 0` guard to
  a literal `1.0`, so `rel_vol` is 1.0 rather than `na` on those
  symbols. Component 3 therefore awards a flat 5 points (the `>= 1.0`
  tier) and Gate 4 never fires, because `volume_is_dead` tests
  `rel_vol < rvol_gate` and MARKET's `rvol_gate` is exactly 1.0.
  Residual effects, both minor and confined to context-only symbols:
  the score ceiling is ~82 instead of 105 (5 of 25 on Component 3, 2 of
  5 on 5C, which needs `rel_vol > 1.5`), and the 1.0-vs-1.0 tie is a
  knife-edge — **raising MARKET's `rvol_gate` above 1.0 would silently
  give NDQ and TSX a permanent −25**. Worth a code comment at line 549
  before anyone tunes that value.
- **Fixed 1% `pnlTarget` means something different on every symbol.**
  1% is a big directional day on SPX, a normal swing on NVDA, and noise
  on NG1!. Current BUY/SELL win rates are therefore not comparable
  across the watchlist — they largely measure how big 1% is relative to
  each instrument. This is the same problem as #4 above, reached from a
  different direction.

### Unifying approach: normalize by measured volatility, not ticker name

Almost every hardcoded threshold exists to compensate for how volatile
an instrument is — which `atr_14` already measures. Define
`atr_pct = daily_ATR(14) / close * 100` (SPX ~0.8%, IBM ~1.3%, NVDA
~3.5%, SPCX ~8%, GC ~1%, NG ~4%) and express percent-based thresholds
as multiples of it:

| Today (static %)     | Proposed              |
|----------------------|-----------------------|
| `vwap_h_limit = 1.8` | `k_h * atr_pct`       |
| `vwap_e_limit = 3.5` | `k_e * atr_pct`       |
| `pnlTarget = 1.0`    | `k_t * atr_pct`       |

The k-multipliers vary far less across asset classes than the raw
percentages do. The profile table survives but is demoted to setting
k-values and clamps rather than absolute levels. New tickers then work
without being added to any list.

### Three layers

1. **Volume guard + rescale.** `vol_available` detector; when false skip
   Component 3, disable Gate 4's RVOL leg, let 5C award full points on
   release-alone, and rescale the remaining components to 100 so NDQ's
   72 means the same as SPX's 72.
2. **ATR normalization.** `pnlTarget`, then VWAP limits, then retiring
   the SuperTrend ticker table in favour of `atr_pct` buckets.
3. **Market/sector context (advisory).** Traded symbols currently have
   no awareness of the indices being watched alongside them. Pull SPX
   regime (trend, ADX, squeeze, % from open) via `request.security` with
   `lookahead_off`, plus SOXX relative strength for AMD/MU/NVDA/SOXX.
   Benchmark routing: SPX for US equities (it has volume, so a full
   regime read is available — NDQ adds little); TSX:TSX for SHOP
   (trend-only, no volume); **no benchmark for CL/NG/GC/SI**, which
   should read `n/a` rather than show a correlation that isn't there.
   Display an alignment counter (0–3: market trend agrees, sector trend
   agrees, RS positive) with no score impact initially.

## Sequencing Plan (2026-08-16)

Two constraints drive the order. First, an ATR-scaled `pnlTarget`
**redefines what counts as a win**, so it must land *before* the
re-baseline this document already calls for — otherwise that baseline is
collected against a ruler that is about to change and has to be thrown
away. Second, the bottleneck is live bars, not coding time: every phase
that changes signal behavior needs its own measurement window.

**Phase 0 — Instrument first, change nothing.** (Idea #5.) Add
expectancy alongside win rate: average win %, average loss %, expectancy
per signal, resolved at the same point the existing `buy_results` /
`sell_results` arrays resolve. Once Phase 2 changes the target
definition, win rate is non-comparable across that boundary while
expectancy stays meaningful — without this there is no way to tell
whether Phase 2 helped or just moved the goalposts. Bump
`DASHBOARD_MAX_ROWS` 28 → 34 here, once (expectancy ~2 rows, context ~4
later); re-run checklist #8 and #11. Success criterion: no signal
behavior moves at all.

**Phase 1 — Volume guard + rescale.** (Layer 1.) Every traded symbol has
volume, so this is provably incapable of touching the baseline — the
only change in the plan that is safe to land during a measurement
window. Verify SPX/SOXX/NVDA scores are unchanged and NDQ/TSX show
`NO VOL` with a rescaled score.

**Phase 2 — ATR-scaled `pnlTarget`, alone.** (Idea #4 / Layer 2.)
Nothing else in this phase — not VWAP limits, not the SuperTrend table.
This is a *metric* change, not a *behavioral* one; bundling it with any
behavioral change makes the two inseparable in the data. Show the
derived target in the dashboard and label text so it can be
sanity-checked per symbol (checklist #10). **Then collect the
re-baseline**, now against a target that means the same thing on IBM,
NVDA, NG and SOXX.

**Phase 3 — Opening-range exclusion.** (Idea #2.) First behavioral
change, measured against the new ruler. The entire chop-loss cluster on
the reviewed chart sits inside 09:30–10:30, it touches no scoring logic,
and it is better-evidenced than the remaining Layer 2 work — so it goes
ahead of it.

**Phase 4 — ATR-normalized VWAP limits.** (Layer 2.) Contained entirely
within Component 2, so blast radius is one scoring component.

**Phase 5 — Retire the SuperTrend ticker table.** (Layer 2.) Highest
blast radius in the plan: it changes trend classification itself, so
every flip, signal, gate reading and win rate moves at once. Last for
that reason, not because it is hard. Also the largest upside, since
eight watchlist symbols are currently defaulted. Verify checklist #7 and
#16 plus a per-symbol flip-count comparison.

**Phase 6 — Market/sector context, advisory.** (Layer 3.) Zero
behavioral impact, so it needs no measurement window of its own — build
it *during* the waiting periods of Phases 2–5. It is the only part of
the plan that parallelizes.

**Phase 7 — Enforcement decisions.** Once alignment values have been
watched against real outcomes: feed the counter into `minScore` and/or
implement #1 (hard regime gate before signal generation). Both are the
same architectural move — converting an advisory reading into a block —
so decide them together.

**Deferred — #3 (pyramiding / re-entry).** Still the biggest lever for
the missed-trend failure mode, and still the biggest redesign: breaking
strict alternation reworks `pnl_entry_price`, the win-rate arrays and
the P&L-history system simultaneously. Revisit after Phase 5 data. It
may also matter less by then, since Phases 3 and 5 attack the same
problem from the entry-timing side.

### Known risks in this ordering

- Phases 2–5 are serialized behind measurement windows, making this a
  multi-week plan. To compress, 3 and 4 could share one window (different
  mechanisms, both low-risk) at the cost of attribution between them.
  Phase 5 should **not** be batched — it needs to be seen in isolation.
- Phase 5 can partially undo gains from Phases 3–4 by changing flip
  timing underneath them. Unavoidable given it is foundational; it is the
  argument for keeping expectancy instrumented throughout so this is
  visible rather than inferred.
- Adaptive thresholds are less legible than a static table. Mitigate by
  showing the *derived* values on the dashboard (ATR%, effective VWAP
  limit, effective target), which is arguably better transparency than
  the table since it shows what is actually in effect.

## Next Step

Phase 0 — add expectancy instrumentation and bump `DASHBOARD_MAX_ROWS`
to 34, with no behavioral change. Then Phase 1 (volume guard), then
Phase 2 (ATR-scaled target) **before** re-measuring win rate with the
2026-08-15 defaults (Risk Gates ON, EMA Slow Period 30) on the SOXX
2-min chart — so that re-baseline is collected once, against the final
target definition, rather than against the fixed 1% ruler it is about to
replace.
