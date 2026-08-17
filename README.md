# SuperLazyTrade

**Systematic Intraday Momentum Trading Indicator for TradingView**
*Pine Script v6 | By @dmandrey*

---

## System Overview

### Core Components (100 Point Scoring)

| # | Component | Points | Notes |
|---|-----------|--------|-------|
| 1 | EMA Cascade | up to 20–30 pts | Profile-adaptive; always scores against EMA 9/20/50 |
| 2 | VWAP Value | up to 15–25 pts | Profile-adaptive; distance from session VWAP |
| 3 | Volume Intensity | 25 pts | RVOL vs Time-of-Day or Rolling 20-bar baseline |
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

**Gates ON (default/recommended):** Penalties subtracted from final score before signal generation.
**Gates OFF:** Warnings display in dashboard but do not affect score.

### Signal Anchors

| Anchor | Trigger | Best For |
|--------|---------|----------|
| EMA Cross (default) | EMA9 crosses slow EMA (20 or 30) | NVDA, TSLA, fast movers |
| SuperTrend | SuperTrend band flip | ETFs, trending markets |
| SMA | Price clears a slow SMA by an ATR buffer | Choppy instruments, fewer/higher-conviction entries |

**SMA anchor detail.** Slowest of the three. Instead of a raw price/SMA cross — which whipsaws badly intraday — the state only flips when price fully clears an ATR-scaled buffer band, and is held inside it:

```
close > SMA + (buffer x ATR-14)  -> BULLISH
close < SMA - (buffer x ATR-14)  -> BEARISH
inside the band                  -> hold previous state
```

SMA length is set per asset profile in AUTO mode, with a MANUAL override:

| Profile | SMA Length |
|---------|-----------|
| STOCK | 70 |
| FUTURES | 70 |
| ETF / FUND | 90 |
| MARKET INDEX | 90 |
| CRYPTO | 120 |

Buffer defaults to `0.25 x ATR`. Set it to `0` for a raw price/SMA cross (many more flips); raise it toward `1.0` for rarer, later entries. The band is drawn on the chart in grey so you can see the zone that is suppressing flips.

> The AUTO lengths and the default buffer are reasoned starting points, not values calibrated against live results. Compare flip frequency across anchors using the Extended Metrics "Bars From Flip" row before trusting them.

---

## Quick Start

### Installation
1. Copy `SuperLazyTrade.pine`
2. Open TradingView → Pine Editor → Paste → Save → Add to Chart

### Recommended Settings
- **Chart:** 2-minute timeframe
- **Assets:** NVDA, TSLA, SPY, QQQ, SOXX, Silver Futures
- **Signal Anchor:** EMA Cross
- **EMA Slow Period:** 30 (smoother, default) or 20 (faster, more whipsaw risk)
- **Signal Timing:** Score mode
- **Min Score:** 50 BUY / 50 SELL
- **Risk Gates:** ON (default — suppresses chop/squeeze/low-liquidity entries; see `roadmap.md`)
- **Relative Volume Mode:** Time-of-Day (default)
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
- Only trade signals within 1-2 bars of generation (or constrain this formally via `Max Bars From Flip` in Score mode)
- Verify VWAP alignment matches direction
- Confirm volume spike (>1.2x minimum for stocks)
- Avoid signals during active squeeze (Gate 1 warning)

### Risk Management
- Two-loss rule per instrument per day
- Close all positions by 15:45 EST (no overnight holds)
- Rotate instruments if conditions deteriorate

### P&L Tracking
- Entry resets on every new BUY/SELL signal (there's only ever one signal per held direction, thanks to strict alternation — see Key Design Decisions)
- **Fixed % target:** ±1.0% default (adjustable); fires PROFIT/LOSS exit labels and alerts
- Position state is flattened at the 9:30 RTH open (when the session filter is on), so an overnight gap can't fire an exit against yesterday's entry price

### Win-Rate Tracking
- **BUY Win Rate / SELL Win Rate:** every signal is scored when the next opposite-direction signal closes it — win only if the ±target was reached during the entry, loss otherwise (stop hit, or no target ever hit); tracked as separate rolling windows
- Independent of `Enable P&L Exit Signals` — that toggle only hides the PROFIT/LOSS labels and alerts; win rates keep resolving either way

---

## Key Design Decisions

**Non-repainting:** All signals confirmed on `barstate.isconfirmed`. Daily ATR uses `lookahead_off`, so the ATR Fuel reading climbs through the session (it feeds Gate 3, not just the display) but a historical bar always recomputes to the value it had live — no signal ever repaints.

**Dual-anchor unification:** After anchor selection, all downstream logic uses `is_bull`/`is_bear`/`trendUp`/`trendDown`. The EMA slow period affects cross detection only — Component 1 always scores against EMA20.

**Signal blocking (strict alternation):** Once a BUY fires, no further BUY can fire — in either Trend or Score mode — until a SELL actually fires, regardless of how many trend flips happen in between. Flags only clear on the opposite signal firing or at RTH session open (only when the session filter is on). P&L tracking keeps running the whole time, unaffected by this gate.

**Gate vs advisory:** `gateOn = true` (default) subtracts gate penalties from score, suppressing chop/squeeze/low-liquidity entries. `gateOn = false` shows gate warnings only. Component 5C (Squeeze Release) is always active regardless of gate mode.

**Time-of-Day RVOL:** Removes the U-shaped intraday volume bias of a trailing SMA by comparing each bar to the historical average for its own bar-slot.

**Entry freshness:** `maxBarsFromFlip` (Score mode only) prevents chasing a trend long after the flip that triggered it.

See `CLAUDE.md` for full architectural detail, gotchas, and the compile-check checklist.

---

## Notes

- TradingView Pro plan or higher required for 2-minute charts
- Designed for US market hours (9:30–16:00 EST); the RTH filter is ET-hardcoded — disable it for crypto or 24h futures
- Educational reference only — no guarantee of trading results
- Test thoroughly on paper before live trading

---

*Trade smart. Trade lazy. Trade systematically.*
