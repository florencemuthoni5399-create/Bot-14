# v2.3.2 — keep news refresh alive and fail over immediately

- Fixed a stale-calendar bug: a Deriv WebSocket close no longer cancels the independent news refresh and watchdog timers. They keep refreshing through Deriv reconnects.
- Corrected `NEWS_RETRY_COUNT` parsing so `0` is honored instead of being changed to the old default by JavaScript's `||` behavior.
- Set the sample default to zero primary retries so a rate-limited ForexFactory-compatible feed falls through immediately to Finance Calendar.
- Kept the news gate fail-closed if both sources fail or the last successful calendar ages out.
- Added the latest calendar-fetch error to the dashboard to make fallback failures visible.

The Finance Calendar fallback was checked against its current documented endpoint and returned a valid event response during this update. The deployed Render service still needs this package and `NEWS_RETRY_COUNT=0` applied before the fix takes effect there.

# v2.3.1 — match the available EUR/USD contract duration

- Changed the default Deriv contract duration from 5 minutes to 15 minutes to match the duration shown in the user's Deriv EUR/USD trading screen.
- Kept the strategy's 5-minute entry candles and 15-minute trend-regime candles unchanged; contract duration and signal timeframes are separate settings.
- Updated the Render environment example and deployment guide. Set `DURATION_VALUE=15` in Render because service environment variables override package defaults.

This is an execution/configuration alignment, not evidence of profitability. Deriv's proposal response remains the authority on whether a quote is available and its current payout.

# v2.3.0 — pullback continuation and quote-aware payout gate

- Added `STRATEGY_MODE=PULLBACK` as the default, alongside `CONFLUENCE` (the prior indicator logic) and `SIMPLE` (legacy tick direction).
- Pullback mode builds 15m trend regime bars from completed 5m EURUSD bars and checks EMA alignment, directional DI, and ADX.
- 5m entries require a prior EMA pullback, a completed-bar confirmation beyond the prior bar, and a maximum ATR extension. The entry is rechecked after a Deriv proposal arrives.
- Added proposal-before-buy execution. Each accepted trade records quoted ask, total payout, and quote break-even.
- MEASURE reports now include a 95% Wilson lower win-rate bound and a strategy fingerprint. Old/mismatched reports and untagged historical trades cannot unlock a different strategy.
- TRADE mode checks each quote's break-even against the matching MEASURE sample's 95% lower win-rate bound plus `PAYOUT_EDGE_BUFFER_PCT`.
- Added Deriv historical candle warmup and configurable absolute paths for the report/trade log, suitable for a Render persistent disk.
- Updated Render deployment instructions, environment examples, and package version.

This strategy is a testable hypothesis, not a claim of profitability. Run MEASURE on demo first and review the report before changing modes.

# v2.2 patch notes — EURUSD measurement stability

## Fixed in this build

- Added a resilient economic-news gate with:
  - the existing ForexFactory-compatible feed as primary;
  - Finance Calendar as an automatic fallback;
  - retries with backoff;
  - `Retry-After` support for HTTP 429 responses;
  - retention of the last successful calendar during temporary fetch failures;
  - fail-closed behavior when there has never been a successful calendar load or the cached calendar becomes stale.
- Kept `NEWS_FAIL_CLOSED=true` by default. A failed calendar must not silently permit trading.
- Added support for multiple common calendar response shapes and UTC timestamp fields.
- Added conservative EUR/USD currency inference for fallback calendar records when a source omits a direct currency field.
- Skip-reason counting now occurs at actual decision points instead of repeatedly every few seconds while the same gate is active.
- Confluence evaluation now occurs only on completed candles. Mid-candle ticks no longer inflate `WARMUP` or gate skip counters.
- Removed `CANDLE_TICKS=60` from the example configuration so the intended 5-minute time-based candles are used by default.
- Martingale remains disabled in MEASURE and hard-capped at 2 levels in TRADE.

## Important

The fallback calendar is provided by Finance Calendar. Its API documentation requests visible attribution when its data appears in an app; the dashboard therefore includes a small source link. See https://www.financecalendar.com/api/.

This build is for demo measurement. It does not guarantee profitability and should not be moved to a real account merely because the measurement report reaches its threshold.


## v2.2.1 — news refresh reliability fix

- Added a 15-second timeout to each news HTTP request so a hung primary/fallback request cannot block future refreshes.
- Changed news refresh scheduling from a fixed interval to a sequential refresh loop, preventing an in-flight request from permanently blocking the next scheduled refresh.
- A valid calendar response with zero matching EUR/USD events now counts as a successful refresh; the bot no longer becomes stale solely because the filtered result is empty.
- Failed refreshes still retain the previous good calendar and fail closed when that calendar becomes stale.
- Added `NEWS_REQUEST_TIMEOUT_MS` to `.env.example`.
- Kept 5-minute candles, MEASURE flat-$1 testing, fail-closed news protection, and the 2-level martingale cap unchanged.

## v2.2.2 — news scheduler hardening
- Replaced chained news refresh scheduling with an independent fixed-interval scheduler.
- Added a 5-minute watchdog that forces a refresh when the normal cadence is missed.
- Default news refresh cadence is now 30 minutes (capped at 60 minutes).
- Preserved fail-closed behavior and the existing primary/fallback sources.
