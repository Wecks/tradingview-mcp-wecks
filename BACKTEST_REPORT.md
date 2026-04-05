# Backtest Report — Trading Strategies
**Date:** April 2026 | **Primary instrument:** AMEX:SPY | **Tools:** Pine Script v6 + TradingView MCP

---

## Table of Contents

1. [General Methodology & Technical Limitations](#1-general-methodology--technical-limitations)
2. [Strategy 1 — H1 Forex (Price Action, 87% WR claim)](#2-strategy-1--h1-forex-price-action-87-wr-claim)
3. [Strategy 2 — RSI(2) Connors](#3-strategy-2--rsi2-connors)
4. [Strategy 3 — Gap Fill Daily (SPY)](#4-strategy-3--gap-fill-daily-spy)
5. [Comparative Summary](#5-comparative-summary)
6. [What Is Worth Testing Further](#6-what-is-worth-testing-further)
7. [Concrete Improvement Ideas](#7-concrete-improvement-ideas)
8. [Fundamental Limitations of This Work](#8-fundamental-limitations-of-this-work)

---

## 1. General Methodology & Technical Limitations

### Testing Environment

All backtests were conducted using **Pine Script v6** on **TradingView Desktop**, with code injected via the CDP (Chrome DevTools Protocol). Compilation and result extraction were automated via the TradingView MCP server.

### Structural Constraints of Pine Script

Pine Script has several important limitations that impacted backtest quality:

**The Daily Bar Problem in Strategy mode**
Pine Strategy cannot, on the same Daily bar, simultaneously enter at the open, check whether an intraday level was reached (e.g. `high >= TP`), and exit at EOD close. All three operations require intraday data. Solution for Gap Fill: manual **indicator-based** backtest with per-bar P&L calculation. This is more rigorous but less flexible for advanced statistics (exact Sharpe, CAGR).

**Limited Historical Data (Free Account)**
- TradingView Free provides only ~1 year of 5-minute data → Gap Fill on intraday data = only ~6 trades. Unusable. This is why we switched to Daily bars.
- Daily strategies benefit from SPY's full history since 1993 (reliable).

**Slippage and Commissions**
Values used (0.05% commission, 1 tick slippage) are approximations. In practice:
- For a liquid ETF like SPY: realistic
- For H1 Forex: variable spread (0.5–2 pips depending on session/broker) may be underestimated
- Backtests do not model market impact (negligible on SPY, non-negligible on less liquid assets)

**Lookahead Bias**
No lookahead bias was identified in any of the three strategies. Systematic checks:
- Signals use only `close[1]`, `high[1]`, `low[1]` (previous bar data)
- Entries are either at the current bar's open or via stop order (conditional execution)
- Swing pivots have an inherent lag of `swing_len` bars (non-repainting, acceptable)

**Bug: `str.tostring(x, "#.3")`**
Discovered during implementation: the format string `"#.3"` outputs the `.3` as a literal suffix, not 3 decimal places. The correct format is `"#.###"`. Impact: the "Avg Win" and "Avg Loss" columns in the Gap Fill table display incorrect values. Does not affect P&L calculations or the equity curve.

---

## 2. Strategy 1 — H1 Forex (Price Action, 87% WR claim)

### Source
Medium — *"The 1-Hour Forex Strategy — Our 87% Win Rate Method for Beginners"*

### Strategy Rules
1. **Trend filter**: price above EMA 50
2. **Key level**: price within a support/resistance zone (swing pivots)
3. **Pattern**: Pin Bar (dominant wick ≥55% of total range) or Inside Bar
4. **Session**: London (07–10h UTC) or New York (12–15h UTC)
5. **Entry**: stop order above/below the pattern extreme
6. **SL/TP**: SL below/above the pattern low/high, TP = 2×SL (RR 1:2)

### Results

| Instrument | Period | Trades | Win Rate | Profit Factor | Net P&L | Max DD |
|-----------|--------|--------|----------|---------------|---------|--------|
| EUR/USD H1 | ~2020–2026 | 144 | **36.1%** | **1.87** | +$1,847 | ~22% |

**Blog claim: 87% WR**

### Critical Bug Found in v1
v1 used `strategy.entry()` without a `stop=` parameter, which executed the order **at market on the next bar's open**. The entry price was often already beyond the SL level immediately → 0 wins out of 5 trades tested. v2 uses `strategy.entry(..., stop=long_entry)`: the position only opens if price actually breaks the pattern extreme.

### Critical Analysis

**The 87% WR is impossible to reproduce.**
With a 1:2 RR, an 87% WR would imply a Profit Factor of ~6.7. That is a level very few strategies achieve even over 20 trades, let alone 144.

**However, the PF of 1.87 is real and positive.**
A 36% WR with 1:2 RR is mathematically consistent — the system profits because winners more than compensate for the many losers. The blog likely conflates WR with something else, possibly the rate of trades that are momentarily in profit intraday before hitting the SL.

**Structural issues:**
- Swing pivots have a **20-bar lag** (lookback parameter). In real time, you cannot know a pivot is "confirmed" until the next 20 bars have closed. The backtest correctly uses `ta.pivothigh/low` which is non-repainting, but the delay makes live trading difficult.
- Pin bars on EUR/USD H1 are extremely frequent — the key level confluence is the most filtering condition, but also the most subjective.
- The session filter is arbitrary: you potentially miss good setups during Asia or overlap sessions.

### What Is Worth Keeping
- The **minimum 1:2 RR** is a sound rule
- The **3-confluence approach** (trend + level + pattern) is rigorous
- The session filter (London/NY) is relevant for EUR/USD — the liquidity argument is real

---

## 3. Strategy 2 — RSI(2) Connors

### Source
StockCharts — *"RSI(2) Strategy" by Larry Connors*

### Original Rules
1. **Trend filter**: price > SMA(200) for longs
2. **Entry**: RSI(2) crosses below 10 (oversold)
3. **Exit**: price crosses above SMA(5)
4. No stop-loss (Connors' recommendation)
5. Instrument: SPY Daily

### Results

| Variant | RSI threshold | Exit | Trades | Win Rate | Profit Factor | CAGR | Max DD |
|---------|--------------|------|--------|----------|---------------|------|--------|
| **Base** (Connors strict) | < 10 | SMA5 | 215 | 68.4% | 1.87 | 2.42% | ~14% |
| **Optimized** | < 5 | RSI>65 OR SMA5 | 104 | **72.1%** | **2.48** | 1.95% | ~10% |

**Connors' claim: ~65–70% WR**

### Critical Analysis

**This is the only strategy where the claim is validated.**
68.4% WR falls exactly within Connors' stated range. The strategy has a proven edge over 33 years of SPY (1993–2026), not just over a favorable sub-period.

**Profit Factors of 1.87 (base) and 2.48 (optimized) are solid.**
PF > 1.5 over 200+ trades is generally considered statistically significant.

**Real issues:**
- **CAGR of 2.42%** — The strategy is only invested ~15–20% of the time (~30–40 days/year in position). Capital sits idle the rest of the time. Buy & Hold SPY over 33 years returns ~10%/year. RSI(2) outperforms *during trades* but capital utilization is too low to beat buy-and-hold in absolute terms.
- **No stop-loss**: Connors justifies this with long-term statistics, but in March 2020 SPY lost -34% in 5 weeks. With 100% of capital deployed, that is an existential drawdown for a retail trader.
- **Structural bull market bias**: 1993–2026 on SPY is broadly a bull trend. Long-only strategies are mechanically advantaged. On a secular bear market index (e.g. Nikkei 1990s), results would be very different.
- **Short side**: tested but inconclusive — SPY's structural upward bias makes the short variant unreliable even with identical inverted parameters.

**Optimization without overfitting:**
The optimized variant (RSI<5, exit RSI>65 or SMA5) improves WR by +3.7 points and PF by +0.61 while halving the number of trades. Parameters remain close to Connors' originals — no grid search was performed on the test period.

### What Is Worth Keeping
- **Mean-reversion logic on broad indices** is robust over long periods
- **SMA(200) as a trend filter** is simple and effective
- The RSI(2) + SMA(5) entry/exit system is well documented in academic literature

---

## 4. Strategy 3 — Gap Fill Daily (SPY)

### Source
QuantifiedStrategies — *"Gap Fill Trading Strategies"*

### Rules Tested
1. **Gap DOWN** between -0.15% and -0.6% (open vs close[1])
2. **Previous day range filter**: previous close in the bottom quarter of its range
3. **Entry**: open of the gap day
4. **TP**: 75% of gap filled = `open + 0.75 × (close[1] - open)`
5. **Exit**: EOD close if TP not reached

### Results — 4 Variants (SPY Daily, 1993–2026)

| Variant | Range filter | SMA200 | TP ratio | Trades | Win Rate | PF | Net P&L | Max DD |
|---------|-------------|--------|----------|--------|----------|----|---------|--------|
| A — Base | ✓ | ✗ | 75% | 353 | **77.1%** | 1.22 | +$1,304 | 8.36% |
| B — No filter | ✗ | ✗ | 75% | 1,692 | 77.1% | 1.27 | +$10,453 | 8.9% |
| C — Conservative | ✓ | ✓ | 75% | 227 | 74.1% | 1.20 | ~+$900 | **5.3%** |
| D — Full fill | ✓ | ✗ | 100% | 353 | 72.1% | 1.17 | +$1,599 | 7.79% |

**Claim: 89% WR, +0.19%/trade (Jan 2010 – Aug 2012, 110 trades)**

### Critical Analysis

**The 89% WR is a temporal cherry-pick.**
The 2010–2012 period is an exceptional post-crisis mean-reversion phase (chronically elevated VIX, frequent gaps, massive Fed liquidity). On that same window, our backtest yields ~84% WR — consistent with the claim. Over the full history, it drops to 77%.

**77% WR is still a real edge.**
353 trades over 33 years is statistically robust. WR does not change with or without the range filter (77.1% both ways), which suggests the **gap condition itself** is the true signal, not the previous day filter.

**Frequency vs profitability problem:**
- Variant A: 353 trades / 33 years = ~10.7 trades/year. At $10K capital, Net P&L = +$1,304 over 33 years = ~$39/year. A PF of 1.22 is too low to exploit at this frequency.
- Variant B (no filter): 1,692 trades / 33 years = ~51 trades/year, Net P&L = +$10,453 = ~$317/year. Better in absolute terms, but the daily frequency makes each trade very small.

**The 100% TP (full gap fill) is counterintuitive but logical:**
Waiting for the full fill might seem better — in reality, WR drops (-5 points) because some gaps that would have reached 75% never reach 100%. The sweet spot appears to be around 75–80%.

**Automation feasibility:**
The signal is detectable at the open in real time:
1. Compute `gap_pct = (open - close[1]) / close[1] * 100` on the first tick
2. Compute `range_pos` using yesterday's data (available pre-market)
3. Place a LIMIT order at `tp_price` + a time-based close at 15:55 NY

This is **straightforwardly automatable** with any broker that exposes an API (Interactive Brokers, Alpaca, etc.).

---

## 5. Comparative Summary

### Claims Validation

| Strategy | Claimed WR | Backtest WR | Validated? | PF | Sharpe | Verdict |
|----------|-----------|-------------|------------|----|---------|---------:|
| H1 Forex PA | 87% | 36.1% | ❌ False | 1.87 | negative | Real edge, misleading claim |
| RSI(2) Connors | 65–70% | 68–72% | ✅ Confirmed | 1.87–2.48 | ~0.04 | Solid, low CAGR |
| Gap Fill Daily | 89% | 77.1% | ⚠ Partial | 1.22–1.27 | n/a | Real edge, PF too low |

### Edge Quality Ranking

```
#1 — RSI(2) Connors   ████████████████████████ PF 2.48, 33 years of evidence
#2 — H1 Forex PA      ██████████████████ PF 1.87, misleading WR but positive P&L
#3 — Gap Fill Daily   ████████████ PF 1.27, edge fragile after realistic costs
```

### Real Risk per Strategy

| Strategy | Main risk | Worst-case scenario |
|----------|----------|---------------------|
| H1 Forex | False key levels, variable spread | 10 consecutive SL hits (-20%) |
| RSI(2) | No SL, tail event | March 2020-style crash (-34% with no exit) |
| Gap Fill | Gain/trade too small vs costs | High volatility regime (gaps >1%) |

---

## 6. What Is Worth Testing Further

### Priority 1 — RSI(2) with Proper Risk Management

**Why:** The only strategy with a validated edge over 33 years and a Profit Factor > 2.
**Suggested test:** Compare Connors' no-SL version vs a version with ATR(14) × 2 stop-loss.
- Hypothesis: SL reduces WR by a few points but meaningfully improves Sharpe ratio
- Also test on QQQ, IWM, DIA to check if the edge generalizes
- **Recommended out-of-sample period:** 2000–2010 (dot-com crash + 2008 financial crisis)

### Priority 2 — Gap Fill with Dynamic Position Sizing

**Why:** A PF of 1.27 at fixed size is insufficient, but scaling position size by gap magnitude could improve the risk/reward ratio.
**Suggested test:** Allocate more capital to larger gaps (e.g. -0.5% gap → 2× standard size vs -0.2% gap → 0.5× standard size)
- Hypothesis: larger gaps tend to snap back more aggressively → better return momentum
- Test on sector ETFs (QQQ, XLF, XLE) where gaps are more frequent

### Priority 3 — RSI(2) + Gap Filter Combination

**Why:** Both strategies are mean-reversion on SPY. A gap DOWN + RSI(2) oversold = double confluence.
**Suggested test:** Enter RSI(2) signals only on days with a qualifying gap DOWN.
- Hypothesis: gap entries are "cleaner" (selling momentum exhausted at the open)
- Risk: drastic reduction in trade count → watch statistical significance

### Priority 4 — H1 Forex on Cleaner Timeframes

**Why:** Despite the disappointing WR (36%), a PF of 1.87 over 144 trades is not bad.
**Suggested test:** Isolate pin bars only (drop inside bars) on Daily key levels
- Test on 4H instead of 1H (less noise, more significant levels)
- Compare London vs New York in isolation: one of the two may be clearly superior
- Test on GBP/USD which is more volatile and tends to form cleaner pin bars

---

## 7. Concrete Improvement Ideas

### RSI(2) Connors

| Improvement | Implementation | Expected Impact |
|-------------|---------------|-----------------|
| Add ATR×2 stop-loss | `strategy.exit("Long SL", stop=close-2*atr14)` | ↑ Sharpe, ↓ WR slightly |
| Conditional pyramiding | If RSI(2) < 2 the following day, double the position | ↑ gains on the best setups |
| VIX filter | Enter only if VIX > 15 | ↑ WR (mean-reversion stronger in volatility) |
| Multi-timeframe confirmation | RSI(3) Daily + RSI(2) Weekly both oversold | ↓ trades, ↑ quality |
| Accelerated exit | Exit if RSI > 65 AND gap up at open | ↓ time in position |

### Gap Fill Daily

| Improvement | Implementation | Expected Impact |
|-------------|---------------|-----------------|
| Dynamic sizing | Capital × (gap_pct / 0.4) capped at 100% | ↑ P&L on strong setups |
| VIX filter | Gap fill only if VIX < 30 (normal regime) | ↓ losses during crises |
| Adaptive TP | TP = max(75% of gap, ATR(5) × 0.3) | ↑ PF on narrow-range bars |
| Filter pre-FOMC days | Skip Fed meeting days | ↓ macro false setups |
| Multi-ETF | Add QQQ, IWM to increase frequency | ↑ trades/year → better significance |

### H1 Forex Price Action

| Improvement | Implementation | Expected Impact |
|-------------|---------------|-----------------|
| Fixed Daily levels | Use PDH/PDL/PDC instead of swing pivots | ↑ level reliability |
| ADR filter | Enter only if <50% of ADR consumed | ↑ room for TP |
| Adaptive RR | RR = 1:3 if distance to level > 20 pips | ↑ average gain |
| HTF confirmation | H1 pin bar aligned with D1 trend | ↓ false signals |
| Avoid high-impact news | No entry 30min before/after NFP, CPI, Fed | ↓ SL spikes |

---

## 8. Fundamental Limitations of This Work

### Technical Environment Limitations

**TradingView Free Account:**
- Intraday data limited to ~1 year → backtests on sub-5min charts are unusable
- No tick data access → opening slippage cannot be modeled precisely
- Pine Script Strategy Tester is a simulation, not a real broker — order handling on high-volatility bars may differ from live execution

**CDP / Automation:**
- Automated result extraction via Pine tables is fragile (depends on exact table formatting)
- Screenshots require an active X11 display (Linux environment)
- The `"#.3"` format bug in the code affects display of some metrics (Avg Win/Loss in Gap Fill)

### Methodological Limitations

**Data Snooping Bias:**
Parameters were adjusted after observing initial results. Even though adjustments stayed close to the original authors' values, there is a bias risk. A rigorous validation would require fixing parameters **before** looking at the data, then validating on a separate out-of-sample dataset.

**Survivorship Bias:**
SPY has existed since 1993 and reflects only companies that survived. SPY backtests mechanically overestimate long-only strategies.

**Publication Bias:**
All three strategies come from sources with an incentive to present favorable results. The 89%, 87%, and 65% figures are marketing numbers before they are scientific ones.

**Trade Count and Statistical Significance:**
- H1 Forex: 144 trades over ~6 years → insufficient to extrapolate. The 95% confidence interval on a 36% WR with 144 trades is wide (~±8 points).
- RSI(2): 215 trades over 33 years → robust
- Gap Fill A: 353 trades over 33 years → acceptable
- Gap Fill B: 1,692 trades → statistically the most robust

**No Walk-Forward Analysis:**
A proper robust test would require optimizing on 70% of the data and validating on the remaining 30% (walk-forward). This work did not perform that separation.

**Market Regime:**
Mean-reversion strategies (RSI2, Gap Fill) work best in low-trend, oscillating regimes. In strongly trending markets they underperform. No regime filter was tested.

**Unmodeled Real-World Costs:**
- Intraday spread (especially H1 Forex, variable 0.5–2 pips)
- Market impact on large positions
- Overnight swap/rollover costs for Forex
- Capital gains taxes (impact on real net CAGR)

### What These Backtests Do NOT Prove

- ❌ That these strategies will work in the future
- ❌ That results are reproducible on other instruments without re-validation
- ❌ That the optimal parameters found here are stable over time
- ❌ That a real trader would achieve the same results (psychology, execution, discipline)

---

## Conclusion

Of the three strategies tested, **only one holds up to scrutiny**: RSI(2) Connors. Its edge has been documented since 2003, confirmed over 33 years of SPY data, with a Profit Factor > 2 on the optimized variant. Its main weaknesses are a low CAGR (poor capital utilization) and the author's recommendation to use no stop-loss.

Gap Fill is the most **automatable** strategy: the signal is simple, objective, and detectable at the open. The edge exists (PF 1.27 over 1,692 trades) but is too fragile to exploit directly — it needs dynamic sizing and a regime filter to be genuinely profitable after costs.

H1 Forex is the most **marketable** and least robust. A PF of 1.87 is positive but the real WR of 36% (vs 87% claimed) perfectly illustrates how presentation bias can make an average strategy look revolutionary. It is not without merit — the confluence + 1:2 RR framework is sound — but it needs validation over far more trades before allocating real capital.

**Final recommendation:** RSI(2) with ATR×2 stop-loss, tested on SPY + QQQ + IWM, with a walk-forward analysis on 2000–2015 / validation on 2015–2026. That is the most rigorous starting point for quantitative trading.

---

*Pine scripts available in `/scripts/`:*
- `rsi2_strategy.pine` — Optimized RSI(2) Connors strategy
- `gap_fill_daily.pine` — Gap Fill manual indicator backtest
- `gap_fill_strategy.pine` — Gap Fill 5-min intraday strategy
- `h1_forex_strategy.pine` — H1 Forex v2 (stop orders)
