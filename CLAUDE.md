# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Pine Script v6 TradingView indicator** — a single-file intraday momentum trading system. The sole source file is `SuperLazyTrade.pine`. There is no build system, package manager, or test runner; development means editing the `.pine` file and pasting it into TradingView's Pine Editor to compile and validate.

Current version: **V50**

## Development Workflow

1. Edit `SuperLazyTrade.pine`
2. Copy the full file contents
3. Open TradingView → Pine Editor → paste → Save → Add to Chart
4. Compilation errors appear immediately in the Pine Editor console
5. Test on a 2-minute chart with a liquid instrument (NVDA, TSLA, SPY, QQQ, SOXX, Silver Futures)

## Architecture

The script is organized into sequential sections (read top-to-bottom, order matters in Pine Script):

1. **Constants** — Gate penalty values, dashboard sizing (`DASHBOARD_MAX_ROWS = 25`)
2. **Inputs** — All user-configurable parameters grouped by function:
   - `grp_anchor`: Signal Anchor (SuperTrend / EMA Cross) + EMA slow period
   - `grp_st`: SuperTrend ATR mode (AUTO / MANUAL) + manual ATR/Factor overrides
   - `grp_sig`: Signal timing, quality filter, min scores, RTH session filter, gate enforcement
   - `grp_htf`: HTF timeframe filter (5 or 15 min)
   - `grp_pnl`: P&L exit signals, target %, tracking mode, signal P&L history
   - `grp_success`: Success rate tracking, rolling window size
   - `grp_vis`: Dashboard visibility, extended metrics, circles, background
3. **Adaptive SuperTrend params** — AUTO mode selects ATR length and factor based on `syminfo.type` and ticker; MANUAL overrides. See [Auto SuperTrend Config Table](#auto-supertrend-config-table).
4. **Asset profile assignment** — Detects `syminfo.type` via a `switch` expression → sets `profile_name` and all threshold variables used throughout scoring. See [Profile Parameters Table](#profile-parameters-table).
5. **Core indicator calculations** — EMAs (9, `emaSlow`=user-choice 20 or 30, 20 fixed for scoring, 50), VWAP, ATR(14), RSI(14), ADX(14,14), MACD(12,26,9), relative volume (vs SMA-20), HTF EMA9/20 via `request.security`
6. **Market regime detection** — Squeeze state (BB(20,2) inside KC(20,1.5×ATR)), squeeze release + bias, stretch factor (EMA distance + trend move since flip), velocity override (ADX>35 rising 3 bars), ATR fuel gauge (session range vs daily ATR-14)
7. **Dual-anchor trend classification** — SuperTrend vs EMA Cross mode; `is_bull`/`is_bear`/`trendUp`/`trendDown` unify both anchors behind a single interface. `trend_start_price` updated on every `trendUp`/`trendDown`.
8. **Scoring engine** — 5 components summed to `raw_score` (capped at 100). Max theoretical total = 105 across all profiles. See [Scoring Components](#scoring-components).
9. **Risk gates** — 5 gates calculate penalties; applied to `raw_score` → `final_score` only when `gateOn = true`; always shown as warnings regardless. See [Risk Gates](#risk-gates).
10. **Signal generation** — Non-repainting; `barstate.isconfirmed` guards all flip conditions. Alternating BUY/SELL enforced via `b_fired`/`s_fired` flags. Flags reset when anchor direction changes (`is_bull` vs `is_bull[1]`), not just on flip bars. RTH filter via `time(timeframe.period, "0930-1600:23456")`.
11. **P&L tracking** — Two independent systems: (a) live dashboard P&L using `pnl_entry_price`/`pnl_direction` with Last Signal or First Signal reset logic; (b) signal-to-signal `signal_pnl` history shown on labels.
12. **Success rate tracking** — Rolling arrays (`buy_results`, `sell_results`) capped at `maxSignalsToTrack`; updated on PROFIT/LOSS exits first (to prevent double-count), then on new signals.
13. **Visuals** — Conditional plots: SuperTrend line (green/red) or EMA9 dynamic line (green/red) based on active anchor; flip circles at transition bars; BUY/SELL labels anchored to active line price.
14. **Dashboard** — `table.new` at `position.bottom_right` with `DASHBOARD_MAX_ROWS = 25`; rendered only on `barstate.islast`. Row overflow protected by `if row >= DASHBOARD_MAX_ROWS: break` guard in gate detail loop.
15. **Alerts** — Four `alertcondition` calls: BUY, SELL, PROFIT, LOSS.

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

**Component 4 — ADX Strength (scales with profile `adx_min`):**
- 15pts: ADX ≥ 1.6× `adx_min` AND rising (2-bar)
- 12pts: ≥ 1.6× OR (≥ 1.3× AND rising)
- 9pts: ≥ 1.3× OR ≥ `adx_min`
- 6/3/0: moderate/weak/choppy

**Component 5 — Momentum Confluence (3 sub-components, max 20):**
- 5A MACD (8pts): MACD line same sign as trend direction
- 5B RSI (7pts): regime-aware — bull wants RSI >60 rising; bear wants RSI <45 falling
- 5C Squeeze Release (5pts): sqz_release + RVOL >1.5 → 5pts; release alone → 2pts

**Setup quality rating** (applied to `final_score` after any gate enforcement):
- ⭐⭐⭐ EXCELLENT: `final_score ≥ threshold + 30`
- ⭐⭐ STRONG: `≥ threshold + 15`
- ⭐ GOOD: `≥ threshold`
- ⚠️ WEAK: below threshold

---

## Risk Gates

5 gates total. In **Advisory mode** (`gateOn = false`, default): shown in dashboard but not applied. In **Enforcement mode** (`gateOn = true`): penalties subtract from `final_score`; all use `math.max(..., 0)` floor before the final clamp `[0, 100]`.

| Gate | Trigger | Penalty | Constant |
|------|---------|---------|----------|
| 1 — Squeeze | BB(20,2) inside KC(20,1.5×ATR) | −30 | `SQUEEZE_PENALTY` |
| 2 — Stretch Factor | Unified: MAX(EMA stretch, trend stretch) > `ext_scale` | −10 or −20 | `STRETCH_MODERATE/EXTREME_PENALTY` |
| 3 — ATR Fuel | Session range > `fuel_max`% of daily ATR-14 AND no velocity override | −25 | `FUEL_PENALTY` |
| 4 — Liquidity | RVOL < `rvol_gate` AND ADX < 90% of `adx_min` | −25 | `LIQUIDITY_PENALTY` |
| 5 — HTF Counter-Trend | HTF EMA9/20 direction opposes signal direction (when `htfEnabled`) | −20 | `HTF_PENALTY` |

**Stretch Factor detail:** `stretch_factor = max(ema_stretch, trend_stretch)` where `ema_stretch = |close - ema20| / ATR(14)` and `trend_stretch = |close - trend_start_price| / ATR(14)`. Moderate penalty when `> ext_scale`; extreme when `> ext_scale × 1.5`. Velocity override (`ADX > 35` rising 3 bars) bypasses Gate 3 only.

---

## Profile Parameters Table

All threshold variables are assigned once per bar based on `syminfo.type`:

| Profile | `vwap_h_limit` | `vwap_e_limit` | `adx_min` | `rvol_gate` | `ext_scale` | `fuel_max` | `ema_max` | `vwap_max` |
|---------|----------------|----------------|-----------|-------------|-------------|------------|-----------|------------|
| MARKET INDEX 🏦 | 0.7% | 1.5% | 25 | 1.0× | 2.0 ATR | 80% | 22 | 23 |
| ETF / FUND 📊 | 1.0% | 2.0% | 18 | 1.0× | 2.2 ATR | 75% | 20 | 25 |
| STOCK 🚀 | 1.8% | 3.5% | 20 | 1.2× | 3.0 ATR | 85% | 20 | 25 |
| FUTURES ⚡ | 1.5% | 3.0% | 15 | 0.6× | 2.5 ATR | 90% | 30 | 15 |
| CRYPTO 🪙 | 3.0% | 6.0% | 28 | 0.6× | 4.0 ATR | 95% | 25 | 20 |
| DEFAULT | 1.5% | 3.0% | 22 | 0.8× | 2.5 ATR | 85% | 20 | 25 |

`syminfo.type == "fund"` maps to FUND; `"futures"` to FUTURES; `"crypto"` to CRYPTO; `"stock"` to STOCK; anything else (index, forex, unknown) to MARKET.

---

## Auto SuperTrend Config Table

Selected when `atrMode = "AUTO"` (default):

| Instrument | Ticker Match | ATR Length | Factor |
|------------|-------------|------------|--------|
| High-vol stocks | NVDA, TSLA | 8 | 2.2 |
| Blue-chip stocks | AAPL, MSFT, AMZN, GOOGL | 10 | 2.5 |
| Generic stock | (other stocks) | 10 | 2.5 |
| Semiconductor ETFs | SOXX, SMH | 10 | 2.5 |
| Broad ETFs | QQQ, SPY, IWM | 12 | 2.8 |
| Generic ETF | (other funds) | 12 | 2.8 |
| Natural Gas | NG* | 8 | 2.2 |
| Silver | SI* | 12 | 2.8 |
| Gold | GC* | 12 | 2.8 |
| Crude Oil | CL* | 10 | 2.5 |
| Index Futures | ES*, NQ* | 12 | 3.0 |
| Generic futures | (other futures) | 10 | 2.5 |
| Crypto | (all) | 12 | 3.5 |
| Other / Index | (fallback) | 14 | 3.0 |

Futures matching uses first 2 characters of `syminfo.ticker`.

---

## Key Design Decisions

**Profile-adaptive scoring:** `ema_max` and `vwap_max` vary by asset type (e.g., FUTURES gets `ema_max=30`, `vwap_max=15`), so the raw 100-point maximum is assembled differently per profile. All profiles sum to 105 before the cap. When modifying component weights, check all five profiles.

**Dual-anchor unification:** After anchor selection, all downstream logic uses `is_bull`, `is_bear`, `trendUp`, `trendDown` — never `st_*` or `ema_*` directly. The EMA slow period (`emaSlowPeriod` = 20 or 30) affects cross detection but NOT Component 1 scoring, which always uses `ema20`. Score mode and Trend mode are both gated through the same `is_bull`/`is_bear` flags.

**Non-repainting guarantee:** Signals only fire on `barstate.isconfirmed`. The daily ATR and HTF EMAs use `request.security(..., lookahead_off)` to prevent lookahead.

**Gate vs advisory mode:** Setting `gateOn = false` (default/recommended) shows gate warnings in the dashboard but does NOT subtract from `final_score`. Component 5C (Squeeze Release) is always active regardless of gate mode — it was moved from a gate bonus in V48B precisely to ensure consistent scoring.

**Signal blocking:** `b_fired` and `s_fired` prevent consecutive same-direction signals. Flags reset when the anchor direction changes (`is_bull` vs `is_bull[1]`), not just on flip bars — this was the V48H fix for EMA Cross mode. When `signal_buy` fires: `b_fired := true`, `s_fired := false` (and vice versa). RTH session open resets both flags so the first RTH bar is never blocked by overnight carryover.

**Two P&L tracking systems (independent):**
- *Live dashboard P&L* — tracks position from `pnl_entry_price`. "Last Signal" mode resets entry on every new signal; "First Signal" mode resets only on direction change, so compounding same-direction signals track the full run.
- *Signal P&L History* — shown on BUY/SELL labels; calculates `%` move from `last_signal_price` to current close using the previous signal's direction. Completely separate state from dashboard P&L.

**Success rate tracking:** PROFIT/LOSS exits are captured *before* new-signal processing on the same bar to prevent double-counting. Results stored in rolling `array<bool>` capped at `maxSignalsToTrack` using FIFO (`array.shift` on overflow).

**Dashboard row budget:** `DASHBOARD_MAX_ROWS = 25` (rows 0–24). With all options enabled + 5 active gates, the dashboard uses up to ~24 rows. The gate detail loop has an explicit `if row >= DASHBOARD_MAX_ROWS: break` guard to prevent runtime overflow.

**VWAP grace zone asymmetry:** Bull grace zone tolerance = `vwap_h_limit × 0.2`; bear grace zone = `vwap_h_limit × 0.1`. Bears are held to half the tolerance — intentional, reflecting that short-side entries near VWAP carry more reversion risk.

---

## Version Naming

The script file is `SuperLazyTrade.pine` (version-agnostic name). The current version is V50. Version identifiers (e.g., V49A) follow major=significant feature additions, minor letter=incremental fixes. The full changelog is documented in the block comments at the top of the file.

Key version milestones:
- **V48B** — Breakout bonus moved from Gate 5 to Component 5C (always active); gate count settled at 4
- **V48D** — Adaptive SuperTrend AUTO mode added
- **V48F** — Dual-anchor system (SuperTrend + EMA Cross)
- **V48H** — EMA Cross signal blocking fix (direction-change reset, not flip-bar reset)
- **V48I** — Continuous EMA9 line (no gaps)
- **V49A** — EMA slow period selector (20 or 30)
- **V49B** — HTF counter-trend filter (Gate 5) added; `ema_stretch` and `trend_stretch` unified via intraday ATR
- **V50** — Bug fixes: dashboard row overflow guard, gate floor protection, session init, redundant flag assignments
