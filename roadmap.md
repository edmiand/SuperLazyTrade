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

## Next Step

Re-measure win rate with the new defaults (Risk Gates ON, EMA Slow
Period 30) on the same SOXX 2-min chart. If the missed-trend problem
persists, revisit #1 (regime/chop hard gate) and #2 (opening-range
exclusion) as the next cheapest fixes, before the larger #3 redesign.
