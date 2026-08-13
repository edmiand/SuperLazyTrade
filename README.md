# SuperLazyTrade V51G

**Systematic Intraday Momentum Trading Indicator for TradingView**
*Pine Script v6 | By @dmandrey*

---

## What's New in V51

The V51 series was a design-review pass. Each phase defaults to the exact V50 behavior (opt-in via a new input) except where noted. Two phases were later reverted after live testing found no value: Phase 4 (Time-of-Day Filters) and Phase 5 (HTF Counter-Trend, which also removed the older V49B HTF Gate 5 it built on) — see Version History.

- **V51A — Time-of-Day Relative Volume:** `rvolMode` (default `Time-of-Day`) compares each bar's volume to the same bar-slot's average across prior sessions, removing the open/lunch volume-shape bias a trailing SMA has. Falls back to the legacy Rolling 20-bar method automatically when session history is thin.
- **V51B — ATR-Based Exits + Time Stop:** `exitMode` (default `Fixed %`, preserves V50) adds an `ATR-Based` alternative — stop/target frozen at entry as a multiple of intraday ATR. Independent `useTimeStop`/`timeStopBars` (default off) forces an exit after N bars with the real P&L sign recorded as win/loss.
- **V51C — Split Exit vs Reversal Win-Rate:** Success tracking is split into `*_exit_results` (target/stop/time-stop) and `*_reversal_results` (closed by an opposite signal) so the win rate no longer mixes two incompatible outcome definitions.
- **V51F — Entry Freshness Constraint:** `maxBarsFromFlip` (default 0 = disabled) constrains Score-mode entries to fire only within N bars of the most recent anchor flip. Trend mode is unaffected (it fires on the flip bar by construction).

---

## System Overview

### Core Components (100 Point Scoring)

| # | Component | Points | Notes |
|---|-----------|--------|-------|
| 1 | EMA Cascade | up to 20–30 pts | Profile-adaptive; always scores against EMA 9/20/50 |
| 2 | VWAP Value | up to 15–25 pts | Profile-adaptive; distance from session VWAP |
| 3 | Volume Intensity | 25 pts | RVOL vs Time-of-Day or Rolling 20-bar baseline (V51A) |
| 4 | ADX Strength | 15 pts | Trend strength, profile-scaled thresholds, rising bonus |
| 5 | Momentum Confluence | 20 pts | MACD (8) + RSI (7) + Squeeze Release (5) |

Raw scores are capped at 100. EMA and VWAP weights vary by asset profile so all profiles sum to 105 before the cap.

### Risk Gates (Advisory or Enforcement)

| Gate | Penalty | Trigger |
|------|---------|---------|
| 1. Squeeze | −30 | Bollinger Bands inside Keltner Channels |
| 2. Stretch Factor | −10/−20 | Price extended from EMA20 or flip price |
| 3. ATR Fuel | −25 | Session range > profile limit % of daily ATR |
| 4. Liquidity | −25 | RVOL low AND ADX choppy |

**Gates OFF (default/recommended):** Warnings display in dashboard but do not affect score.
**Gates ON:** Penalties subtracted from final score before signal generation.

### Signal Anchors

| Anchor | Trigger | Best For |
|--------|---------|----------|
| EMA Cross (default) | EMA9 crosses slow EMA (20 or 30) | NVDA, TSLA, fast movers |
| SuperTrend | SuperTrend band flip | ETFs, trending markets |

---

## Quick Start

### Installation
1. Copy `SuperLazyTrade.pine`
2. Open TradingView → Pine Editor → Paste → Save → Add to Chart

### Recommended Settings
- **Chart:** 2-minute timeframe
- **Assets:** NVDA, TSLA, SPY, QQQ, SOXX, Silver Futures
- **Signal Anchor:** EMA Cross
- **EMA Slow Period:** 20 (fast) or 30 (smoother)
- **Signal Timing:** Score mode
- **Min Score:** 50 BUY / 50 SELL
- **Risk Gates:** OFF
- **Relative Volume Mode:** Time-of-Day (default)
- **Exit Mode:** Fixed % (default) or ATR-Based
- **Max Bars From Flip:** 0 (disabled by default; Score mode only)

---

## Signal Quality Tiers

| Rating | Condition | Suggested Sizing |
|--------|-----------|-----------------|
| ⭐⭐⭐ EXCELLENT | Score ≥ threshold + 30 | Full size |
| ⭐⭐ STRONG | Score ≥ threshold + 15 | 75% |
| ⭐ GOOD | Score ≥ threshold | 50% |
| ⚠️ WEAK | Score < threshold | Skip |

*Example with minScore=50: EXCELLENT ≥80, STRONG 65–79, GOOD 50–64, WEAK <50*

---

## Asset Profiles

The indicator auto-detects asset type (via `syminfo.type`) and applies optimized thresholds:

| Profile | EMA max | VWAP max | ADX min | RVOL gate | Fuel max | Stretch (ATR) |
|---------|---------|----------|---------|-----------|----------|---------------|
| STOCK | 20 | 25 | 20 | 1.2× | 85% | 3.0 |
| ETF/FUND | 20 | 25 | 18 | 1.0× | 75% | 2.2 |
| FUTURES | 30 | 15 | 15 | 0.6× | 90% | 2.5 |
| CRYPTO | 25 | 20 | 28 | 0.6× | 95% | 4.0 |
| MARKET INDEX *(index, forex, unknown)* | 22 | 23 | 25 | 1.0× | 80% | 2.0 |

SuperTrend ATR length and factor also auto-select per instrument in AUTO mode (see `CLAUDE.md` for the full ticker table).

---

## Trading Rules

### Entry Discipline
- Only trade signals within 1-2 bars of generation (or constrain this formally via `Max Bars From Flip` in Score mode, V51F)
- Verify VWAP alignment matches direction
- Confirm volume spike (>1.2x minimum for stocks)
- Avoid signals during active squeeze (Gate 1 warning)

### Risk Management
- Two-loss rule per instrument per day
- Close all positions by 15:45 EST (no overnight holds)
- Rotate instruments if conditions deteriorate

### P&L Tracking Modes
- **Last Signal (default):** Entry resets on every new BUY/SELL — tracks per-signal P&L
- **First Signal:** Entry resets only on direction change — tracks full directional run
- **Fixed % target:** ±1.0% default (adjustable); fires PROFIT/LOSS exit labels and alerts
- **ATR-Based exits (V51B):** stop/target frozen at entry as a multiple of intraday ATR; optional Time Stop forces an exit after N bars

### Win-Rate Tracking (V51C)
- **Exit results** (BUY/SELL Exits): outcome of PROFIT/LOSS/TIME STOP exits only
- **Reversal results** (BUY/SELL Reversal, Extended Metrics only): outcome of positions closed by an opposite signal instead of a target/stop
- Kept as separate rolling windows so the win rate can actually validate parameter changes

---

## Key Design Decisions

**Non-repainting:** All signals confirmed on `barstate.isconfirmed`. Daily ATR uses `lookahead_off` — display can flicker intrabar but no signal ever repaints.

**Dual-anchor unification:** After anchor selection, all downstream logic uses `is_bull`/`is_bear`/`trendUp`/`trendDown`. The EMA slow period affects cross detection only — Component 1 always scores against EMA20.

**Signal blocking:** `b_fired`/`s_fired` prevent consecutive same-direction signals in Score mode only. Flags reset when anchor direction changes (`is_bull` vs `is_bull[1]`), not just on flip bars. Trend mode fires directly off flip bars and never checks these flags.

**Gate vs advisory:** `gateOn = false` (default) shows gate warnings but does not subtract from score. Component 5C (Squeeze Release) is always active regardless of gate mode.

**Time-of-Day RVOL (V51A):** Removes the U-shaped intraday volume bias of a trailing SMA by comparing each bar to the historical average for its own bar-slot.

**Entry freshness (V51F):** `maxBarsFromFlip` (Score mode only) prevents chasing a trend long after the flip that triggered it.

---

## Version History

- **V51G** (2026-08): HTF Counter-Trend filter removed entirely (Gate 5, input group, dashboard row) — no observable difference in signal behavior found in testing; risk gates back to 4
- **V51F** (2026-08): Entry Freshness Constraint — `maxBarsFromFlip` limits Score-mode entries to N bars after an anchor flip
- **V51E** (2026-08): HTF Counter-Trend Penalty → optional Hard Veto (`htfMode`) — *removed in V51G*
- **V51D** (2026-08): Time-of-Day Filters — added, then removed as low-value (no `⏰` block-open/lunch/cutoff inputs remain)
- **V51C** (2026-08): Split Exit vs Reversal win-rate tracking
- **V51B** (2026-08): ATR-Based Exits + Time Stop
- **V51A** (2026-08): Time-of-Day Relative Volume
- **V50**: Hard-coded RTH session filter (9:30–4:00 PM ET); dashboard/gate bug fixes
- **V49B**: Fixed stretch_factor ATR scale mismatch; added HTF Trend Filter (Gate 5) — *removed in V51G*
- **V49A**: User-selectable EMA slow period (20 or 30)
- **V48I**: Continuous EMA9 line (no gaps between trend segments)
- **V48H**: EMA Cross mode signal blocking fix; anchor-aware label positioning
- **V48G**: Dynamic anchor visualization (hide inactive anchor)
- **V48F**: Dual-anchor system — SuperTrend or EMA Cross user selection
- **V48E**: Removed early-session detection logic
- **V48D**: Adaptive SuperTrend parameters by asset type (AUTO/MANUAL)
- **V48B**: Moved breakout bonus to Component 5C (always-active scoring)
- **V48A**: Removed Component 6 (Candle Structure) and Gate 6 (Late Entry)
- **V45** (Earlier): Unified Stretch Factor gate (mean reversion + trend exhaustion)
- **V44** (Earlier): Hybrid success rate tracking with rolling window

See `CLAUDE.md` for full architectural detail, gotchas, and the compile-check checklist.

---

## Notes

- TradingView Pro plan or higher required for 2-minute charts
- Designed for US market hours (9:30–16:00 EST); the RTH filter is ET-hardcoded — disable it for crypto or 24h futures
- Educational reference only — no guarantee of trading results
- Test thoroughly on paper before live trading

---

*Trade smart. Trade lazy. Trade systematically.*
