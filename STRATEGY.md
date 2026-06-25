# How AXIS Decides to Trade

This document explains the exact decision logic behind both agents. Nothing here is hidden or hand-waved, every rule below maps directly to a function in `trading-agent.html`.

## 1. Perceive: what the agent looks at

Every decision cycle, the agent pulls live BTC/USDT data from Bitget's public API (price + 1H candles) and computes three things in-browser:

| Indicator | Settings | What it tells the agent |
|---|---|---|
| EMA50 / EMA100 | 50 and 100-period exponential moving averages | The macro trend direction. This is deliberately a longer-term, slower pair (not EMA20/50) so the agent doesn't get faked out by short-term noise. |
| RSI | 14-period | Whether price is stretched (oversold below 35, overbought above 65), used for mean-reversion timing |
| MACD | 12, 26, 9 | Momentum and trend shifts, via the line/signal crossover |

Code: `computeAllIndicators()`, `calcMomentum()`

## 2. Decide: trend sets direction, momentum times the entry

**Step 1, trend filter:** Compare EMA50 to EMA100.
- EMA50 above EMA100 → uptrend → only LONG entries are considered
- EMA50 below EMA100 → downtrend → only SHORT entries are considered (Futures agent only)

**Step 2, momentum score:** RSI and MACD each contribute -1, 0, or +1 to a momentum score:
- RSI < 35 → +1 (oversold, bullish) · RSI > 65 → -1 (overbought, bearish)
- MACD bullish crossover → +1 · MACD bearish crossover → -1

**Step 3, combine:**
- **Futures agent**: enters LONG if (uptrend AND score ≥ 1), enters SHORT if (downtrend AND score ≤ -1)
- **Spot agent** (stricter, since it can't short and kept getting whipsawed on weak signals): enters LONG only if the uptrend is clearly confirmed (EMA50 more than 0.4% above EMA100, not a marginal crossover) AND score ≥ 2 (RSI and MACD both agree). In a downtrend, Spot does nothing and waits in cash.

Code: `futuresDecision()`, `spotDecision()`

## 3. Execute: simulated paper trade

- Position size: 20% of that agent's own paper balance
- Spot can only ever be long or flat. Futures can be long, short, or flat.
- Only one open position per agent at a time
- Every trade is logged with timestamp, pair, direction, price, quantity, and balance change (see the CSV trade logs in this repo)

Code: `openPosition()`, `closePosition()`

## 4. Manage risk: the same rule on every single trade, no exceptions

- **Stop-loss**: 3% against the entry price
- **Take-profit**: 6% in favor of the entry price
- **Simulated fee**: 0.1% per side, deducted from every trade
- **Trend-flip exit**: if the macro trend flips against an open position before SL/TP is hit, the agent closes the position anyway rather than holding through a reversal

Code: `checkRisk()`

## Why this design

A long-only strategy loses by default in a downtrend, that's not bad luck, it's structural. AXIS fixes that by giving Futures the ability to go short when the trend is down, while keeping Spot honest about a real constraint it actually has (you can't short a spot wallet). The EMA50/100 pair was chosen specifically to reduce whipsaw trades compared to faster moving averages, verified by comparing backtest trade frequency and win rate before and after the change.

Every number in this document (3%, 6%, 35, 65, 0.4%, 20%) is a fixed rule applied identically across the full backtest window, not tuned after the fact to make one specific historical run look better. Run "Run Backtest" on the live demo at any time to see this logic execute against whatever BTC actually did in the most recent 30 days.
