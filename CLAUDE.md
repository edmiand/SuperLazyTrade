# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Pine Script v6 TradingView indicator** — a single-file intraday momentum trading system. The sole source file is `SuperLazyTrade.pine`. There is no build system, package manager, or test runner; development means editing the `.pine` file and pasting it into TradingView's Pine Editor to compile and validate.

## Development Workflow

1. Edit `SuperLazyTradeV49A.pine`
2. Copy the full file contents
3. Open TradingView → Pine Editor → paste → Save → Add to Chart
4. Compilation errors appear immediately in the Pine Editor console
5. Test on a 2-minute chart with a liquid instrument (NVDA, TSLA, SPY, QQQ, SOXX, Silver Futures)

## Architecture

The script is organized into sequential sections (read top-to-bottom, order matters in Pine Script):

1. **Constants** — Gate penalty values, dashboard sizing
2. **Inputs** — All user-configurable parameters grouped by function (`grp_anchor`, `grp_st`, `grp_sig`, `grp_pnl`, `grp_success`, `grp_vis`)
3. **Adaptive SuperTrend params** — AUTO mode selects ATR length and factor based on `syminfo.type` and ticker; MANUAL overrides
4. **Asset profile assignment** — Detects `syminfo.type` → sets `profile_name` and threshold variables (`vwap_h_limit`, `adx_min`, `rvol_gate`, `ext_scale`, `fuel_max`, `ema_max`, `vwap_max`) used throughout scoring
5. **Core indicator calculations** — EMAs (9, slow-period, 20, 50), VWAP, ATR, RSI, ADX, MACD, relative volume, Bollinger Bands, Keltner Channels
6. **Market regime detection** — Squeeze state (BB inside KC), stretch factor (EMA distance + trend move since flip), ATR fuel gauge (session range vs daily ATR)
7. **Dual-anchor trend classification** — SuperTrend vs EMA Cross mode; `is_bull`/`is_bear`/`trendUp`/`trendDown` unify both anchors behind a single interface
8. **Scoring engine** — 5 components summed to `raw_score` (capped at 100):
   - Component 1: EMA Cascade (up to `ema_max` pts, profile-adaptive)
   - Component 2: VWAP Value (up to `vwap_max` pts, profile-adaptive)
   - Component 3: Volume Intensity (25 pts fixed)
   - Component 4: ADX Strength (15 pts fixed)
   - Component 5: Momentum = MACD (8) + RSI (7) + Squeeze Release (5)
9. **Risk gates** — 4 gates calculate penalties; applied to `raw_score` → `final_score` only when `gateOn = true` (enforcement mode); always displayed as warnings
10. **Signal generation** — Non-repainting; `barstate.isconfirmed` guards; alternating BUY/SELL enforced via `b_fired`/`s_fired` flags reset on anchor direction flip
11. **P&L tracking** — Two parallel systems: live dashboard P&L (entry price resets per `pnlTrackingMode`), and signal-to-signal P&L history (`showSignalPnL`)
12. **Success rate tracking** — Rolling arrays (`buy_results`, `sell_results`) capped at `maxSignalsToTrack`; updated on new signals and on PROFIT/LOSS exits
13. **Visuals** — Conditional plots (SuperTrend line vs EMA9 dynamic line), flip circles, BUY/SELL labels, PROFIT/LOSS labels
14. **Dashboard** — `table.new` at `position.bottom_right`; rendered only on `barstate.islast`
15. **Alerts** — Four `alertcondition` calls (BUY, SELL, PROFIT, LOSS)

## Key Design Decisions

**Profile-adaptive scoring:** `ema_max` and `vwap_max` vary by asset type (e.g., FUTURES gets `ema_max=30`, `vwap_max=15`), so the raw 100-point maximum is assembled differently per profile. When modifying component weights, check all five profiles.

**Dual-anchor unification:** After anchor selection, all downstream logic uses `is_bull`, `is_bear`, `trendUp`, `trendDown` — never `st_*` or `ema_*` directly. The EMA slow period (`emaSlowPeriod` = 20 or 30) affects cross detection but NOT Component 1 scoring, which always uses `ema20`.

**Non-repainting guarantee:** Signals only fire on `barstate.isconfirmed`. The daily ATR uses `request.security(..., lookahead_off)` to prevent lookahead.

**Gate vs advisory mode:** Setting `gateOn = false` (default/recommended) shows gate warnings in the dashboard but does NOT subtract from `final_score`. Component 5C (Squeeze Release) is always active regardless of gate mode — it was moved from a gate bonus in V48B precisely to ensure consistent scoring.

**Signal blocking:** `b_fired` and `s_fired` prevent consecutive same-direction signals. Flags reset when the anchor direction changes (`is_bull` vs `is_bull[1]`), not just on flip bars — this was the V48H fix for EMA Cross mode.

## Version Naming

The script file is `SuperLazyTrade.pine` (version-agnostic name). The current version is V49A; the full changelog history is documented in the comments at the top of the file. Version identifiers (e.g., V49A) follow major=significant feature additions, minor letter=incremental fixes.
