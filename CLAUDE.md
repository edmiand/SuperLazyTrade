# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Pine Script v6 TradingView indicator** — a single-file intraday momentum trading system. The sole source file is `SuperLazyTrade.pine`. There is no build system, package manager, or test runner; development means editing the `.pine` file and pasting it into TradingView's Pine Editor to compile and validate.

## Development Workflow

1. Edit `SuperLazyTrade.pine`
2. Copy the full file contents
3. Open TradingView → Pine Editor → paste → Save → Add to Chart
4. Compilation errors appear immediately in the Pine Editor console
5. Test on a 2-minute chart with a liquid instrument (NVDA, TSLA, SPY, QQQ, SOXX, Silver Futures)

## Architecture

The script is organized into sequential sections (read top-to-bottom, order matters in Pine Script):

1. **Constants** — Gate penalty values, dashboard sizing (`DASHBOARD_MAX_ROWS = 28`)
2. **Inputs** — All user-configurable parameters grouped by function:
   - `grp_anchor`: Signal Anchor (SuperTrend / EMA Cross) + EMA slow period
   - `grp_st`: SuperTrend ATR mode (AUTO / MANUAL) + manual ATR/Factor overrides
   - `grp_sig`: Signal timing, quality filter, min scores, RTH session filter, gate enforcement
   - `grp_pnl`: P&L exit signals, target %, signal P&L history
   - `grp_success`: Success rate tracking, rolling window size
   - `grp_vis`: Dashboard visibility, extended metrics, circles, background
3. **Adaptive SuperTrend params** — AUTO mode selects ATR length and factor based on `syminfo.type` and ticker; MANUAL overrides. See [Auto SuperTrend Config Table](#auto-supertrend-config-table).
4. **Asset profile assignment** — A `switch` on `syminfo.type` first computes `asset_category` (STOCK/FUND/FUTURES/CRYPTO/MARKET); a separate `if/else if` chain keyed on `asset_category` then sets `profile_name` and all threshold variables used throughout scoring. See [Profile Parameters Table](#profile-parameters-table).
5. **Core indicator calculations** — EMAs (9, `emaSlow`=user-choice 20 or 30, 20 fixed for scoring, 50), VWAP, ATR(14), RSI(14), ADX(14,14), MACD(12,26,9), relative volume (`rvolMode`: Time-of-Day default, Rolling 20-bar SMA fallback/alternative — see [Time-of-Day Relative Volume](#key-design-decisions))
6. **Market regime detection** — Squeeze state (BB(20,2) inside KC(20,1.5×ATR)), squeeze release + bias, stretch factor (EMA distance + trend move since flip), velocity override (ADX>35 rising 3 bars), ATR fuel gauge (session range vs daily ATR-14)
7. **Dual-anchor trend classification** — SuperTrend vs EMA Cross mode; `is_bull`/`is_bear`/`trendUp`/`trendDown` unify both anchors behind a single interface. `trend_start_price` updated on every `trendUp`/`trendDown`.
8. **Scoring engine** — 5 components summed to `raw_score` (capped at 100). Max theoretical total = 105 across all profiles. See [Scoring Components](#scoring-components).
9. **Risk gates** — 4 gates calculate penalties; applied to `raw_score` → `final_score` only when `gateOn = true`; always shown as warnings regardless. See [Risk Gates](#risk-gates).
10. **Signal generation** — Signals are non-repainting; `barstate.isconfirmed` guards all flip conditions. In "Score" signal-timing mode, alternating BUY/SELL is enforced via `b_fired`/`s_fired` flags; in "Trend" mode, signals fire directly off `trendUp`/`trendDown` and never check these flags — alternation there falls out of the flip-bar semantics instead. Flags reset when anchor direction changes (`is_bull` vs `is_bull[1]`), not just on flip bars. RTH filter via `time(timeframe.period, "0930-1600:23456")`. Score-mode signals can also be constrained to fire only within `maxBarsFromFlip` bars of the anchor flip (default 0 = disabled); Trend mode is unaffected.
11. **P&L tracking** — Two independent systems: (a) live dashboard P&L using `pnl_entry_price`/`pnl_direction`, reset on every new signal; (b) signal-to-signal `signal_pnl` history shown on labels. PROFIT/LOSS exit triggers are Fixed %-only: `current_pnl` vs `±pnlTarget`.
12. **Success rate tracking** — Rolling arrays (`buy_results`/`sell_results`) track PROFIT/LOSS win rate only, capped at `maxSignalsToTrack`.
13. **Visuals** — Conditional plots: SuperTrend line (green/red) or EMA9 dynamic line (green/red) based on active anchor; flip circles at transition bars; BUY/SELL labels anchored to active line price.
14. **Dashboard** — `table.new` at `position.bottom_right` with `DASHBOARD_MAX_ROWS = 28`; rendered only on `barstate.islast`. The gate detail loop is the last section written; it has an explicit `if row >= DASHBOARD_MAX_ROWS: break` guard because it is the only variable-length section. All preceding rows are bounded by design.
15. **Alerts** — Four `alertcondition` calls with plaintext messages: `"BUY"`, `"SELL"`, `"PROFIT"`, `"LOSS"`.

---

## Scoring Components

All 5 components sum to `raw_score`, capped at 100. Component maxes vary by profile but always total 105 before the cap.

| # | Component | Max Points | Varies by Profile? |
|---|-----------|-----------|-------------------|
| 1 | EMA Cascade Alignment | `ema_max` (20–30) | Yes |
| 2 | VWAP Value Anchor | `vwap_max` (15–25) | Yes |
| 3 | Volume Intensity | 25 (fixed) | No |
| 4 | ADX Trend Strength | 15 (fixed) | No |
| 5 | Momentum Confluence | 20 (fixed) | No |

**Component 1 — EMA Cascade (profile-adaptive):**
- Full (`ema_max`): `ema9 > ema20 > ema50` + close on right side of ema9
- Partial (60%): close beyond ema20 but cascade incomplete
- Weak (30%): close beyond ema50 only
- Zero: counter-trend
- Always uses `ema20` (not `emaSlow`) — slow period choice only affects anchor, not scoring

**Component 2 — VWAP Value (profile-adaptive):**
- BOUNCE/REJECTION (full `vwap_max`): within 0.33× `vwap_h_limit` of VWAP, correct side
- HEALTHY (72%): within `vwap_h_limit`, correct side
- EXTENDED (40%): within `vwap_e_limit`, correct side
- REVERSION RISK (0): beyond `vwap_e_limit`, correct side
- GRACE ZONE (20%): slightly wrong side — bull: within 20% of `vwap_h_limit`; bear: within 10% (intentionally stricter)
- WRONG SIDE (0): too far on wrong side

**Component 3 — Volume Intensity:**
- 25pts: RVOL ≥ 2.5× (EXPLOSIVE)
- 20pts: ≥ 2.0× | 15pts: ≥ 1.5× | 10pts: ≥ 1.2× | 5pts: ≥ 1.0× | 0pts: below
- RVOL denominator is `rvolMode`-dependent — point tiers themselves never changed. See [Time-of-Day Relative Volume](#key-design-decisions).

**Component 4 — ADX Strength (scales with profile `adx_min`, nested if/elseif — rising (2-bar) qualifies each tier):**
- 15pts: ADX ≥ 1.6× `adx_min` AND rising | 12pts: ADX ≥ 1.6× `adx_min` and NOT rising
- 12pts: ADX in [1.3×, 1.6×) `adx_min` AND rising | 9pts: same range, NOT rising
- 9pts: ADX in [1.0×, 1.3×) `adx_min` (no rising check)
- 6/3/0: moderate (≥0.8×) / weak (≥0.6×) / choppy (below)

**Component 5 — Momentum Confluence (3 sub-components, max 20):**
- 5A MACD (8pts): MACD line same sign as trend direction
- 5B RSI (7pts): regime-aware — bull wants RSI >60 rising; bear wants RSI <45 falling
- 5C Squeeze Release (5pts): sqz_release + RVOL >1.5 → 5pts; release alone → 2pts

**Setup quality rating** (applied to `final_score` after any gate enforcement):

`threshold` = `minScoreBuy` (bull) or `minScoreSell` (bear) — both are user inputs defaulting to 50, range 0–70. Max 70 ensures ⭐⭐⭐ EXCELLENT (threshold+30) is always reachable.

- ⭐⭐⭐ EXCELLENT: `final_score ≥ threshold + 30`
- ⭐⭐ STRONG: `≥ threshold + 15`
- ⭐ GOOD: `≥ threshold`
- ⚠️ WEAK: below threshold

---

## Risk Gates

4 gates total. In **Advisory mode** (`gateOn = false`, default): shown in dashboard but not applied. In **Enforcement mode** (`gateOn = true`): penalties subtract from `final_score`; all use `math.max(..., 0)` floor before the final clamp `[0, 100]`.

| Gate | Trigger | Penalty | Constant |
|------|---------|---------|----------|
| 1 — Squeeze | BB(20,2) inside KC(20,1.5×ATR) | −30 | `SQUEEZE_PENALTY` |
| 2 — Stretch Factor | Unified: MAX(EMA stretch, trend stretch) > `ext_scale` | −10 or −20 | `STRETCH_MODERATE/EXTREME_PENALTY` |
| 3 — ATR Fuel | Session range > `fuel_max`% of daily ATR-14 AND no velocity override | −25 | `FUEL_PENALTY` |
| 4 — Liquidity | RVOL < `rvol_gate` AND ADX < 90% of `adx_min` | −25 | `LIQUIDITY_PENALTY` |

**Stretch Factor detail:** `stretch_factor = max(ema_stretch, trend_stretch)` where `ema_stretch = |close - ema20| / ATR(14)` and `trend_stretch = |close - trend_start_price| / ATR(14)`. Moderate penalty when `> ext_scale`; extreme when `> ext_scale × 1.5`. Velocity override (`ADX > 35` rising 3 bars) bypasses Gate 3 only.

---

## Profile Parameters Table

All threshold variables are assigned once per bar based on `syminfo.type`. The `switch` expression maps every possible `syminfo.type` value: `"stock"` → STOCK, `"fund"` → FUND, `"futures"` → FUTURES, `"crypto"` → CRYPTO, and **everything else** (index, forex, unknown) → MARKET via the switch default. The MARKET profile is the true catch-all; there is no separate DEFAULT profile in the code.

| Profile | `vwap_h_limit` | `vwap_e_limit` | `adx_min` | `rvol_gate` | `ext_scale` | `fuel_max` | `ema_max` | `vwap_max` |
|---------|----------------|----------------|-----------|-------------|-------------|------------|-----------|------------|
| MARKET INDEX 🏦 *(index, forex, unknown)* | 0.7% | 1.5% | 25 | 1.0× | 2.0 ATR | 80% | 22 | 23 |
| ETF / FUND 📊 | 1.0% | 2.0% | 18 | 1.0× | 2.2 ATR | 75% | 20 | 25 |
| STOCK 🚀 | 1.8% | 3.5% | 20 | 1.2× | 3.0 ATR | 85% | 20 | 25 |
| FUTURES ⚡ | 1.5% | 3.0% | 15 | 0.6× | 2.5 ATR | 90% | 30 | 15 |
| CRYPTO 🪙 | 3.0% | 6.0% | 28 | 0.6× | 4.0 ATR | 95% | 25 | 20 |

---

## Auto SuperTrend Config Table

Selected when `atrMode = "AUTO"` (default). Futures matching uses the first 2 characters of `syminfo.ticker`. Rows with identical ATR/Factor values reflect `else` branches in the code — no extra tuning exists for those sub-groups.

| Instrument | Ticker Match | ATR Length | Factor |
|------------|-------------|------------|--------|
| High-vol stocks | NVDA, TSLA | 8 | 2.2 |
| All other stocks *(incl. AAPL, MSFT, AMZN, GOOGL)* | (any) | 10 | 2.5 |
| Semiconductor ETFs | SOXX, SMH | 10 | 2.5 |
| All other ETFs/funds *(incl. QQQ, SPY, IWM)* | (any) | 12 | 2.8 |
| Natural Gas | NG* | 8 | 2.2 |
| Silver | SI* | 12 | 2.8 |
| Gold | GC* | 12 | 2.8 |
| Crude Oil | CL* | 10 | 2.5 |
| Index Futures | ES*, NQ* | 12 | 3.0 |
| All other futures | (any) | 10 | 2.5 |
| Crypto | (all) | 12 | 3.5 |
| Other / Index / fallback | (all) | 14 | 3.0 |

---

## Key Design Decisions

**Profile-adaptive scoring:** `ema_max` and `vwap_max` vary by asset type (e.g., FUTURES gets `ema_max=30`, `vwap_max=15`), so the raw 100-point maximum is assembled differently per profile. All profiles sum to 105 before the cap. When modifying component weights, check all five profiles.

**Dual-anchor unification:** After anchor selection, all downstream logic uses `is_bull`, `is_bear`, `trendUp`, `trendDown` — never `st_*` or `ema_*` directly. The EMA slow period (`emaSlowPeriod` = 20 or 30) affects cross detection but NOT Component 1 scoring, which always uses `ema20`. Score mode and Trend mode are both gated through the same `is_bull`/`is_bear` flags.

**Non-repainting:** *Signals* are non-repainting — `barstate.isconfirmed` guards every flip condition and all signal assignments. The daily ATR (`atr_14`) uses `request.security(..., lookahead_off)`, which still reads the developing daily bar's live value intrabar — display-only; no signal fires until bar close.

**Gate vs advisory mode:** Setting `gateOn = false` (default/recommended) shows gate warnings in the dashboard but does NOT subtract from `final_score`. Component 5C (Squeeze Release) is always active regardless of gate mode, to keep scoring consistent.

**Signal blocking (strict alternation):** `b_fired`/`s_fired` block a same-direction signal from firing again — in both "Trend" and "Score" signal-timing modes — until the opposite signal actually fires, no matter how many trend flips happen in between. E.g. BUY fires, trend flips bear but SELL's score/quality filter never clears, trend flips bull again → BUY stays blocked. The flags do **not** reset on a trend direction change by itself (`is_bull` vs `is_bull[1]`); they only clear when `signal_buy`/`signal_sell` actually fires (`b_fired := true, s_fired := false` and vice versa) or at RTH session open (`useSessionFilter`-gated, so 24h instruments with the filter off never get a daily reset). Live P&L tracking (`pnl_entry_price`, dashboard P&L, signal-to-signal P&L history) does not consult these flags and keeps accumulating across the held direction regardless.

**RTH filter on 24h instruments:** `time(timeframe.period, "0930-1600:23456")` always evaluates against ET hours regardless of instrument type. The filter does NOT auto-disable for CRYPTO or 24h FUTURES. If `useSessionFilter = true` on those instruments, signals will be blocked outside 9:30–4:00 ET Mon–Fri with no warning. Recommended: disable `useSessionFilter` for crypto and around-the-clock futures contracts.

**Entry freshness constraint:** `maxBarsFromFlip` (0-50, default 0 = disabled) constrains Score-mode entries to fire only within N bars of the most recent anchor flip, via `entry_is_fresh = maxBarsFromFlip <= 0 or bars_since_flip <= maxBarsFromFlip` AND'd into the Score-mode branch only. `bars_since_flip` (`var int`) resets to 0 on the flip bar itself and increments every bar after, tracked alongside the existing `trend_start_price` reset so both stay in sync with the same flip event. Trend mode is deliberately unaffected — it already fires directly on the flip bar by construction, so there's nothing to constrain. This is independent of (and complements) the Stretch gate, which only actually penalizes extension when `gateOn = true`; `maxBarsFromFlip` works regardless of gate mode. Calibrate using the Extended Metrics "Bars From Flip" row.

**Two P&L tracking systems (independent):**
- *Live dashboard P&L* — tracks position from `pnl_entry_price`, reset on every new `signal_buy`/`signal_sell`. Since strict alternation (see Signal Blocking above) means there's only ever one signal per held direction, entry always corresponds to the signal that opened the current position — a separate "First Signal" reset-on-direction-change mode is no longer meaningful and was removed.
- *Signal P&L History* — shown on BUY/SELL labels; calculates `%` move from `last_signal_price` to current close using the previous signal's direction. Completely separate state from dashboard P&L.

**P&L exits are Fixed %-only:** `signal_profit`/`signal_loss` fire when `current_pnl` (computed from `pnl_entry_price`) crosses `±pnlTarget`.

**PROFIT/LOSS-only win-rate tracking:** `buy_results`/`sell_results` track only entries closed by a PROFIT/LOSS target exit (`signal_profit`/`signal_loss`) — win rate no longer mixes in entries closed by the next opposite-direction signal (removed as low-signal noise; reversals just reflect that the position was held until the trend flipped, not that the target/stop calibration was wrong). Each is an independent rolling `array<bool>` capped at `maxSignalsToTrack` via FIFO (`array.shift` on overflow). `calc_win_rate(arr, enabled) => [wins, total, rate]` computes stats for both from one shared read-only function. No double-count guard is needed: `hit_profit`/`hit_loss` are already gated by `pnl_exit_fired`, so `signal_profit`/`signal_loss` each fire exactly once per entry.

**Dashboard row budget:** `DASHBOARD_MAX_ROWS = 28` (rows 0–27). With all options enabled + 4 active gates, the dashboard has one row of headroom below that cap. The gate detail loop is the last section; it has an explicit `if row >= DASHBOARD_MAX_ROWS: break` guard because it is the only variable-length section (all other rows are fixed-count by design). Success Rate Tracking occupies exactly 3 rows (separator + BUY Win Rate + SELL Win Rate) whenever enabled, regardless of Extended Metrics.

**VWAP grace zone asymmetry:** Bull grace zone tolerance = `vwap_h_limit × 0.2`; bear grace zone = `vwap_h_limit × 0.1`. Bears are held to half the tolerance — intentional, reflecting that short-side entries near VWAP carry more reversion risk.

**Time-of-Day Relative Volume:** `rel_vol` (used by Component 3, Gate 4 liquidity, and Component 5C squeeze-release) is `rvolMode`-dependent. Default `"Time-of-Day"` compares current bar volume to the average volume of the *same bar-slot* (bars since session open) across the prior `rvolLookbackDays` completed sessions — this removes the U-shaped intraday volume bias that a trailing SMA has (overstates RVOL near the open, understates it at lunch). History is kept in `array<TodSession>`, where `TodSession` is a UDT wrapping `array<float>` — Pine cannot nest arrays directly (`array<array<float>>` is not supported), so a UDT wrapper is the standard workaround. Session boundaries reuse the existing `is_new_session` (calendar-day change) detector, so the same extended-hours caveat applies as the ATR Fuel gauge. Falls back to the `ta.sma(volume, 20)` method (`"Rolling 20-bar"`) when fewer than 3 historical sessions exist at a slot, or when the user selects Rolling 20-bar explicitly. The active mode is shown inline in the dashboard's Volume row (`TOD`, `20-bar`, or `20-bar*` for an in-flight TOD→fallback).

---

## Pine Script v6 Gotchas

Common failure modes when editing this script:

- **`var` variables** persist across bars and only initialize on bar 0. Do not use `var` for values that must recompute each bar (e.g., profile thresholds). The profile threshold variables (`vwap_h_limit`, `ema_max`, etc.) deliberately omit `var` so they reassign every bar.
- **`request.security` series vs simple:** The daily ATR (`atr_14`) uses `lookahead = barmerge.lookahead_off`. Omitting this causes future-bar lookahead (repainting). Always pass the expression directly — do not pre-compute into a `var` before passing to `request.security`.
- **`na` propagates through arithmetic:** `math.max(na, 0)` returns `na`, not 0. Always guard with `not na(x)` or `nz(x, 0)` before arithmetic involving potentially uninitialized series. This script uses explicit `not na(...)` checks before VWAP and session range calculations.
- **`barstate.isconfirmed` on historical bars:** All historical bars are "confirmed" during replay. `barstate.islast` is true only on the most recent bar. The dashboard uses `barstate.islast`; signals use `barstate.isconfirmed`.
- **`ta.crossover`/`ta.crossunder` are single-bar events:** They return `true` for exactly one bar. No multi-bar guard is needed; however, both require at least 2 bars of history.
- **`array.get` throws on out-of-bounds:** Always check `array.size(arr) > 0` before accessing index 0. The success rate arrays are guarded by `totalTrades_buy > 0` before the win-count loop.
- **`time()` session strings use ET for US equities** but the exchange timezone for other instruments. `"0930-1600:23456"` is hardcoded against ET — this is correct for US stocks but not appropriate for crypto or 24h futures without user awareness (see RTH filter note above).
- **`str.split` includes empty strings** if the delimiter appears at the start/end of the string. `gate_message` is built with a trailing space, so `str.split(gate_message, " ")` always yields a trailing empty token. The word-wrap loop never turns this into a visible row, though: an empty word either merges into the current line as a harmless trailing space (line stays under the 30-char cap) or, if a line break is forced first, becomes `current_line := ""`, which the final `if current_line != "": array.push(...)` guard then excludes. No blank row is ever pushed to `gate_lines`.
- **Arrays cannot nest** — `array<array<float>>` does not compile. To build a 2D/ragged structure (e.g. `TodSession` in the Time-of-Day RVOL feature), wrap the inner array in a user-defined `type` and use `array<TodSession>` instead.
- **Functions cannot reassign a value-type global variable** (`float`/`int`/`bool`/`string`), even one declared with `var` — attempting `pnl_entry_price := close` inside a `set_pnl_entry(dir) =>` function fails to compile with `CE10088 Cannot modify global variable`. Only reference types (`array`/`matrix`/`map`/UDT) can be mutated from inside a function, and only via a parameter (e.g. `add_result(array<bool> arr, ...)` calling `array.push(arr, ...)`). This is why the P&L entry/re-arm logic (`pnl_entry_price`, `pnl_direction`, `pnl_exit_fired`) stays inlined at each of the 8 signal branches instead of being factored into a helper function — wrapping that state in a UDT to allow a helper would touch every read site across the file.

---

## Compile-Check Checklist

No test runner exists. After any edit, verify manually in this order:

1. **Paste into Pine Editor → zero compilation errors** before proceeding
2. **Load NVDA 2-min chart** → confirm dashboard appears at bottom-right
3. **Score ≤ 100** on dashboard at all times (raw_score is capped, but confirm no overflow)
4. **Signal alternation:** let a BUY fire → confirm next BUY is blocked until a SELL fires, including across multiple trend flips where SELL's quality filter never clears (BUY → flip bear, no SELL → flip bull again → still no second BUY)
5. **No double-count on same-bar PROFIT + new signal:** when a PROFIT fires on the same bar as a new signal, confirm success rate increments by 1, not 2
6. **Gate enforcement:** toggle `Enable Risk Gates` ON → confirm score drops when gates are active; toggle OFF → score unchanged but warnings visible
7. **EMA Cross mode:** switch anchor to EMA Cross → confirm EMA9 line appears (not SuperTrend line), flip circles appear at crossover bars
8. **Dashboard row count:** with Success Rate Tracking + all 4 gates active + Extended Metrics ON, confirm no runtime error (row overflow guard working; `DASHBOARD_MAX_ROWS = 28`)
9. **RVOL mode:** toggle `Relative Volume Mode` between `Rolling 20-bar` and `Time-of-Day` on the same chart → confirm the Volume row's RVOL multiplier and mode label (`TOD` / `20-bar`) both change, and that a chart with less than `RVOL Lookback Sessions` of history shows `20-bar*` (fallback) instead of `na` or a stale value
10. **P&L Target:** with `Enable P&L Exit Signals` on, confirm PROFIT/LOSS fires at exactly `±P&L Target (%)` from `pnl_entry_price` and labels show the static target text (e.g. `PROFIT +1.0%`)
11. **Win rate:** with `Enable Success Rate Tracking` on, confirm the dashboard shows `BUY Win Rate`/`SELL Win Rate`; confirm a PROFIT/LOSS exit increments the matching count exactly once and a signal-closed-by-opposite-signal (no PROFIT/LOSS) does not increment either count
12. **Entry freshness:** in Score mode, set `Max Bars From Flip = 5` → confirm no BUY/SELL fires on bar 6+ after a flip (watch the "Bars From Flip" Extended Metrics row to confirm the count itself resets to 0 on each flip bar); confirm Trend mode is unaffected by the same setting; confirm worst case (Success Rate + Extended Metrics + all 4 gates on) still shows no dashboard row overflow with `DASHBOARD_MAX_ROWS = 28`

---

## Version

The script file is `SuperLazyTrade.pine` (version-agnostic name). It is currently baselined at **V1** with no changelog history tracked in the file or in this document — treat the current source as the reference behavior going forward.
