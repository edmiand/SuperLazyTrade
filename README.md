# SuperLazyTrade V48B

**Systematic Intraday Momentum Trading Indicator for TradingView**  
*Pine Script v6 | By @dmandrey | February 2026*

---

## 🎯 What's New in V48B

**Gate 5 (Breakout Bonus) → Component 5C (Squeeze Release Momentum)**

- Breakout quality now **always recognized** in base scoring (regardless of Gates ON/OFF)
- Component 5 expanded: 15 pts → 20 pts max
- Gate count reduced: 5 → 4 (pure risk filters only)
- **Key benefit:** Consistent scoring when trading with Gates OFF

---

## 📊 System Overview

### **Core Components (100 Point Scoring)**
1. **EMA Cascade** (20 pts) - Trend alignment across 9/20/50 EMAs
2. **VWAP Value** (25 pts) - Price positioning vs session anchor
3. **Volume Intensity** (25 pts) - Relative volume confirmation
4. **ADX Strength** (15 pts) - Trend strength measurement
5. **Momentum Confluence** (20 pts) - MACD + RSI + Squeeze Release 🆕

### **Risk Gates (Optional Enforcement)**
1. **Squeeze** (-30) - Range-bound conditions
2. **Stretch Factor** (-10/-20) - Overextension risk
3. **ATR Fuel** (-25) - Volatility exhaustion
4. **Liquidity** (-25) - Dead volume + chop

---

## 🚀 Quick Start

### **Installation**
1. Copy `SuperLazyTradeV48B.pine` code
2. Open TradingView → Pine Editor
3. Paste code → Save → Add to Chart

### **Recommended Settings**
- **Chart:** 2-minute timeframe
- **Assets:** Liquid instruments (NVDA, TSLA, SPY, QQQ, SOXX, Silver Futures)
- **Signal Timing:** "Score" mode (default)
- **Min Score:** 55 for BUY and SELL
- **Risk Gates:** OFF for consistent scoring (recommended for intraday)

### **Signal Quality Filter**
- **⭐ GOOD+:** Show good, strong, excellent signals (recommended)
- **⭐⭐ STRONG+:** High-conviction only
- **⭐⭐⭐ EXCELLENT:** Top-tier setups only

---

## 📈 Signal Quality Tiers

| Rating | Score Range | Action | Position Size |
|--------|-------------|--------|---------------|
| ⭐⭐⭐ EXCELLENT | Threshold +30 | Strong conviction | 100% |
| ⭐⭐ STRONG | Threshold +15-29 | High quality | 75% |
| ⭐ GOOD | Threshold +0-14 | Acceptable | 50% |
| ⚠️ WEAK | Below threshold | Skip | 0% |

*Example with minScore=55: EXCELLENT ≥85, STRONG 70-84, GOOD 55-69, WEAK <55*

---

## 🎓 Trading Rules

### **Entry Discipline**
- ✅ Only trade signals within 1-2 bars of generation
- ✅ Verify VWAP alignment matches direction
- ✅ Confirm volume spike (>1.2x minimum)
- ✅ Check for extended moves (red flag if >5% from entry zone)

### **Risk Management**
- 🛑 Two-loss rule per instrument per day
- 🛑 Close all positions by 15:45 EST (no overnight holds)
- 🛑 Rotate instruments if conditions deteriorate
- 🛑 Skip signals during active squeeze (Gate 1)

### **P&L Tracking**
- **First Signal Mode** (default): Tracks full directional run
- **Last Signal Mode:** Resets on every new signal
- **Target:** ±1.0% default (adjustable)

---

## 🆕 What Changed from V48A

### **Component 5: Momentum (15→20 pts)**
```
V48A:
├─ 5A: MACD (8 pts)
└─ 5B: RSI (7 pts)

V48B:
├─ 5A: MACD (8 pts)
├─ 5B: RSI (7 pts)
└─ 5C: Squeeze Release (5 pts) ← NEW
    ├─ Perfect: +5 (squeeze + vol >1.5x)
    └─ Weak: +2 (squeeze only)
```

### **Gates Simplified (5→4)**
```
V48A: 4 penalties + 1 bonus = mixed architecture
V48B: 4 penalties only = pure risk filters
```

### **Impact**
**Gates OFF (Your Mode):**
- V48A: Breakout ignored in score
- V48B: Breakout always counted (+5)

**Result:** More consistent scoring, no "missing quality" when Gates OFF

---

## 📚 Documentation

### **Included Files**
- `SuperLazyTradeV48B.pine` - Main indicator script
- `V48B_CHANGELOG.md` - Detailed changelog with examples
- `V48B_SCORING_REFERENCE.md` - Complete scoring tables & guidelines
- `README.md` - This file

### **Key Features**
- ✅ **Non-repainting:** All signals confirmed on bar close
- ✅ **Profile-adaptive:** Auto-adjusts for STOCK/ETF/FUTURES/CRYPTO/MARKET
- ✅ **Compile-safe:** Clean Pine Script v6, no errors
- ✅ **Dashboard:** Real-time scoring, gates, P&L tracking
- ✅ **Success rate tracking:** Win/loss stats for BUY and SELL
- ✅ **Alerts:** BUY, SELL, PROFIT, LOSS alerts available

---

## 🔧 System Requirements

- **TradingView:** Pro plan or higher (for 2-min charts)
- **Assets:** Liquid instruments with reliable volume data
- **Session:** US market hours (9:30-16:00 EST) for best results
- **Data:** Real-time or delayed (delay affects signal timing)

---

## ⚠️ Important Notes

### **What This System Is:**
- Systematic momentum quality filter
- Early-entry timing framework
- Risk management discipline enforcer

### **What This System Is NOT:**
- Holy grail / automated money printer
- Substitute for risk management
- Guarantee of profitability

### **Disclaimers**
- Educational reference only
- No guarantee of trading results
- Past performance ≠ future results
- Always use proper position sizing and stops
- Test thoroughly before live trading

---

## 🎯 Philosophy

**"Quality over Quantity"**

SuperLazyTrade prioritizes **setup quality** and **early timing** over signal frequency. The scoring framework systematically identifies high-conviction momentum opportunities while filtering out:
- Late entries (extended moves)
- Range-bound chop (squeeze gate)
- Overextended setups (stretch gate)
- Dead volume conditions (liquidity gate)
- Exhausted volatility (fuel gate)

**Core Principle:** Wait for the right setup, enter early, manage risk systematically.

---

## 📞 Support & Feedback

- **Questions:** Review `V48B_SCORING_REFERENCE.md` for detailed scoring breakdown
- **Issues:** Check compilation in Pine Editor, verify TradingView plan supports features
- **Enhancements:** Test thoroughly before requesting changes - system is battle-tested

---

## 📝 Version History

- **V48B** (2026-02-03): Moved breakout bonus to Component 5C (always-active scoring)
- **V48A** (2026-01-30): Removed Component 6 (Candle Structure) and Gate 6 (Late Entry)
- **V47** (2026-01-29): Removed 2-bar confirmation requirement
- **V45** (Earlier): Consolidated Stretch Factor gate (unified mean reversion + trend exhaustion)
- **V44** (Earlier): Added hybrid success rate tracking
- **V37** (Earlier): Added Squeeze and Liquidity gates

---

## 🚀 Ready to Trade

1. ✅ Upload script to TradingView
2. ✅ Apply to 2-min chart on liquid instrument
3. ✅ Set minScore = 55 (or adjust to preference)
4. ✅ Enable dashboard + extended metrics
5. ✅ Set alerts for BUY/SELL signals
6. ✅ Trade systematically with discipline

**Trade smart. Trade lazy. Trade systematically.** 📈

---

*For detailed scoring breakdown, see `V48B_SCORING_REFERENCE.md`*  
*For complete changelog, see `V48B_CHANGELOG.md`*
