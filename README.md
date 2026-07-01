# SuperLazyTrade V49A

**Systematic Intraday Momentum Trading Indicator for TradingView**  
*Pine Script v6 | By @dmandrey | February 2026*

---

## What's New in V49A

**User-selectable EMA slow period (20 or 30) for EMA Cross anchor**

- EMA 9/20 (default): Responsive, 5-10 signals/session — best for NVDA, TSLA
- EMA 9/30 (smoother): More selective, 3-7 signals/session — best for SOXX, SPY, QQQ
- Dashboard auto-displays active pair (e.g., `EMA Cross (9/30)`)
- Easy A/B testing without code changes

---

## System Overview

### Core Components (100 Point Scoring)

| # | Component | Points | Notes |
|---|-----------|--------|-------|
| 1 | EMA Cascade | up to 20–30 pts | Profile-adaptive; always uses EMA 9/20/50 |
| 2 | VWAP Value | up to 15–25 pts | Profile-adaptive; distance from session VWAP |
| 3 | Volume Intensity | 25 pts | Relative volume vs 20-bar SMA |
| 4 | ADX Strength | 15 pts | Trend strength, profile-scaled thresholds |
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
| EMA Cross (default) | EMA9 crosses slow EMA | NVDA, TSLA, fast movers |
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

The indicator auto-detects asset type and applies optimized thresholds:

| Profile | EMA max | VWAP max | ADX min | RVOL gate | Fuel max |
|---------|---------|----------|---------|-----------|----------|
| STOCK | 20 | 25 | 20 | 1.2 | 85% |
| ETF/FUND | 20 | 25 | 18 | 1.0 | 75% |
| FUTURES | 30 | 15 | 15 | 0.6 | 90% |
| CRYPTO | 25 | 20 | 28 | 0.6 | 95% |
| MARKET INDEX | 22 | 23 | 25 | 1.0 | 80% |

SuperTrend ATR length and factor also auto-select per asset type in AUTO mode.

---

## Trading Rules

### Entry Discipline
- Only trade signals within 1-2 bars of generation
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
- **Target:** ±1.0% default (adjustable); fires PROFIT/LOSS exit labels and alerts

---

## Key Design Decisions

**Non-repainting:** All signals confirmed on `barstate.isconfirmed`. Daily ATR uses `lookahead_off`.

**Dual-anchor unification:** After anchor selection, all downstream logic uses `is_bull`/`is_bear`/`trendUp`/`trendDown`. The EMA slow period affects cross detection only — Component 1 always scores against EMA20.

**Signal blocking:** `b_fired`/`s_fired` prevent consecutive same-direction signals. Flags reset when anchor direction changes (`is_bull` vs `is_bull[1]`), not just on flip bars.

**Gate vs advisory:** `gateOn = false` shows gate warnings but does not subtract from score. Component 5C (Squeeze Release) is always active regardless of gate mode.

---

## Version History

- **V49A** (2026-02-06): User-selectable EMA slow period (20 or 30)
- **V48I** (2026-02-05): Continuous EMA9 line (no gaps between trend segments)
- **V48H** (2026-02-05): EMA Cross mode signal blocking fix; anchor-aware label positioning
- **V48G** (2026-02-05): Dynamic anchor visualization (hide inactive anchor)
- **V48F** (2026-02-05): Dual-anchor system — SuperTrend or EMA Cross user selection
- **V48E** (2026-02-05): Removed early-session detection logic
- **V48D** (2026-02-05): Adaptive SuperTrend parameters by asset type (AUTO/MANUAL)
- **V48B** (2026-02-03): Moved breakout bonus to Component 5C (always-active scoring)
- **V48A** (2026-01-30): Removed Component 6 (Candle Structure) and Gate 6 (Late Entry)
- **V45** (Earlier): Unified Stretch Factor gate (mean reversion + trend exhaustion)
- **V44** (Earlier): Hybrid success rate tracking with rolling window

---

## Notes

- TradingView Pro plan or higher required for 2-minute charts
- Designed for US market hours (9:30–16:00 EST)
- Educational reference only — no guarantee of trading results
- Test thoroughly on paper before live trading

---

*Trade smart. Trade lazy. Trade systematically.*
