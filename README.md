# SynthTrade Pro — EUR/USD Deriv Rise/Fall bot

Node.js 18+ headless bot for Deriv's `frxEURUSD` 15-minute Rise/Fall contracts by default. Entries use 5-minute candles with a 15-minute trend regime. It is not a spot-FX or CFD order manager.

## Strategy modes

- `STRATEGY_MODE=PULLBACK` (default): build completed 5-minute candles, resample them into 15-minute regime candles, then look for a pullback-and-reclaim entry.
- `STRATEGY_MODE=CONFLUENCE`: previous EMA/ADX/ATR/RSI strategy.
- `STRATEGY_MODE=SIMPLE`: previous last-tick-direction signal, retained for comparison only.

Pullback defaults:

- 15m regime: EMA(13)/EMA(34) alignment, +DI/−DI direction agreement, and ADX(14) at or above 18.
- 5m setup: the prior candle touches or comes within 0.2 ATR of EMA(13); the latest completed candle closes back in trend direction beyond the prior candle's high/low and the EMA.
- Entry is skipped if that close is more than 0.75 ATR from EMA(13). The bot rechecks the current spot against this extension limit after the quote arrives.

These are configurable starting values, not evidence of profitability. Historical 5m candles are requested from Deriv at startup to avoid waiting many hours for indicator warmup.

## MEASURE and quote-aware TRADE gate

- `BOT_MODE=MEASURE` is the default. It uses flat `STAKE` (default 1 in the account currency) and disables martingale.
- Each signal first requests a Deriv proposal. The bot records the quote's ask price and total payout with the trade.
- After 500 settled trades, `measure_report.json` records win rate, its 95% Wilson lower bound, payout/breakeven metrics, net profit, and the strategy fingerprint.
- `BOT_MODE=TRADE` requires a completed report for the same symbol, strategy, duration, session/news gates, and risk settings, plus the existing measured EDGE threshold (`TRADE_EDGE_MIN_PCT`, default 1.5%).
- Before each TRADE-mode buy, the bot computes quote break-even as `ask_price / payout × 100`. It buys only when the MEASURE sample's 95% lower win-rate bound is greater than that quote's break-even plus `PAYOUT_EDGE_BUFFER_PCT` (default 0.5 percentage points).
- Old reports and log records without the matching strategy fingerprint are not used to unlock TRADE.

This is a conservative filter, not a guarantee: a confidence bound from historical outcomes can still fail out of sample, and one aggregate win-rate estimate does not capture every market regime or payout condition.

## Risk defaults

- Demo account by default; keep `DERIV_ACCOUNT_TYPE=demo` while measuring.
- Session: 08:00–17:00 UTC. EUR/USD medium-or-higher news blackout: 30 minutes before/after.
- With the default `NEWS_RETRY_COUNT=0`, a failed primary calendar request immediately falls back to Finance Calendar. Refresh timers continue through Deriv WebSocket reconnects; if both feeds fail or the calendar goes stale, the default fail-closed gate pauses trading.
- Daily stop: −3% loss or +1.5% profit relative to the reconstructed UTC day-start balance.
- Three consecutive losses trigger a cooldown.
- Martingale is disabled in MEASURE; in TRADE it is controlled by `MARTINGALE_ENABLED` and capped at two levels.

Do not move to a real account merely because the report passes its gate.

## Local setup

1. Use Node.js 18 or newer.
2. Copy `.env.example` to `.env`.
3. Set `DERIV_APP_ID`, `DERIV_API_TOKEN`, and a strong `DASHBOARD_TOKEN` in your local environment.
4. Keep `DERIV_ACCOUNT_TYPE=demo`, `BOT_MODE=MEASURE`, and `STRATEGY_MODE=PULLBACK`.
5. Run `npm ci`, then `npm start`.

Never commit `.env` or put an API token in source code.

## Render setup

Use a Render **Web Service** with:

- Build command: `npm ci`
- Start command: `npm start`
- Environment: `DERIV_ACCOUNT_TYPE=demo`, `BOT_MODE=MEASURE`, `STRATEGY_MODE=PULLBACK`, and the remaining values from `.env.example`.
- Secrets: set `DERIV_API_TOKEN` and `DASHBOARD_TOKEN` in Render's environment settings, not in Git.

The report and append-only trade log must survive restarts to preserve measurement progress. Attach a Render persistent disk mounted at `/var/data`, then set:

```text
TRADE_LOG_FILE=/var/data/trades.log.jsonl
MEASURE_REPORT_FILE=/var/data/measure_report.json
```

Render supplies `PORT`; the bot respects it. Open the service URL with `?token=<your DASHBOARD_TOKEN>` to view the read-only status page.

## Runtime files

- `trades.log.jsonl`: append-only settled trades, including strategy fingerprint and the proposal quote.
- `measure_report.json`: generated after 500 matching MEASURE trades.

## Source and limitations

The bot uses Deriv's WebSocket API for proposals, buys, ticks, and open-contract updates. It does not backtest or establish that the strategy is profitable. Always validate the new strategy on a demo account and review the live quote, payout, and report before changing modes.