# AXIS: BTC Adaptive Trend + Mean-Reversion Agent
**Bitget AI Base Camp Hackathon S1, Track 1: Trading Agent**

Live demo: https://axie-trading-bot.vercel.app/

## Project Description

**Problem.** Manual crypto trading is reactive and emotional. Traders chase price after it's already moved, and rarely manage risk consistently. Most simple bots are also long-only, so they lose money by default whenever the market is in a downtrend, and bots that react to short-term noise get whipsawed in choppy conditions. There's also no clean way to compare a spot-only strategy against a leveraged/short-capable one side by side.

**Solution.** AXIS runs two autonomous agents in parallel, both built on the same BTC adaptive trend and mean-reversion logic, but constrained to match their real-world instrument:
- **Spot Agent**: long-only, since real spot markets can't short. It waits in cash during downtrends until the next uptrend signal.
- **Futures Agent**: long or short, simulating a USDT-margined perpetual position.

Each one perceives the market, decides a direction, executes a simulated trade, and manages risk automatically with stop-loss and take-profit, with no human in the loop. A combined Portfolio view tracks both side by side.

**Tech approach.** Both agents pull live price and candle data from Bitget's public Spot market API. They compute RSI(14), MACD(12,26,9), and EMA(20/50/100) directly in JavaScript. A macro trend filter (EMA50 vs EMA100, deliberately longer-term than EMA20/50 to avoid whipsawing on short-term noise) decides the only direction each agent is allowed to trade. RSI mean-reversion and MACD momentum then time the entry within that trend. The Futures agent acts on both LONG and SHORT signals; the Spot agent only acts on LONG signals and otherwise stays flat. When running inside Claude.ai, both agents also call the Claude API (Sonnet 4.6) to narrate their reasoning in plain language; outside that environment they fall back to their own rule-based reasoning, so the app always works standalone.

**Strategy loop (Perceive, Decide, Execute, Risk), run independently per agent:**
1. **Perceive**: fetch latest BTC/USDT price and 1H candles from Bitget
2. **Decide**: macro trend (EMA50 vs EMA100) sets the allowed direction; RSI mean-reversion and MACD momentum time the entry within that trend
3. **Execute**: open or close a simulated position sized at 20% of that agent's own paper balance
4. **Risk**: automatic 3% stop-loss and 6% take-profit on every open position, with a 0.1% simulated taker fee per side

## App structure

- **Home**: project overview, track info, and a live combined snapshot of both agents' balances
- **Spot**: long-only agent dashboard (indicators, decision, risk stats, trade log)
- **Futures**: long/short agent dashboard, same layout, simulated perpetual
- **Portfolio**: combined balance, combined equity curve, and a merged trade history tagged by agent

## Instrument

- Spot agent: real BTC/USDT spot rules, long-only
- Futures agent: simulated USDT-margined perpetual (BTC/USDT-PERP equivalent), long or short, 1x notional, no real leverage

Both are paper trading only. No real funds, no real orders, and no API keys with trade permission are used anywhere in this project.

## How to reproduce the backtest

The backtest isn't a static notebook, it's live and reproducible right inside the app:
- Open the live demo link above
- Go to the Spot or Futures tab and click "Run Backtest (30d)"
- This fetches about 30 days of real Bitget 1H candles on the spot and re-simulates that agent's strategy from scratch

The exact logic is in `trading-agent.html`: shared decision math in `calcMomentum()`, the two engines in `spotDecision()` and `futuresDecision()`, the generic backtest runner `runBacktest()`, and position math in `openPosition()`, `closePosition()`, and `checkRisk()`. Judges can read the source directly or re-run it live against current market data at any time. Results will reflect whatever BTC actually did in the prior 30 days at the time of running.

## Trade logs

Each agent has its own "Export Log (CSV)" button. This repo includes:
- `spot-trade-log.csv`
- `futures-trade-log.csv`

Columns:

| Column | Description |
|---|---|
| timestamp | ISO 8601 time of the trade |
| trading_pair | e.g. BTCUSDT |
| direction | BUY (open long), SELL (close long), SHORT (open short), or COVER (close short) |
| price | execution price |
| quantity | position size in base asset |
| account_balance_change | realized P&L or capital movement from this trade |
| account_balance_after | that agent's paper balance immediately after this trade |
| reason | the agent's own reasoning for the action |

## Stack

- Bitget public Spot Market API (ticker and candles), no API key required
- Vanilla JavaScript, single self-contained HTML file, no build step
- Claude Sonnet 4.6 (Anthropic API) for optional live reasoning narration
- Deployed on Vercel

## Run it yourself

No install needed. `trading-agent.html` is a single static file. Open it in any browser, or deploy it to Vercel or Netlify as-is.
