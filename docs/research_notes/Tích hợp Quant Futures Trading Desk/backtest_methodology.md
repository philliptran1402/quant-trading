# Backtesting methodology for a Quant crypto-perpetual research service

## 1. Market data, funding and event-time alignment

### Takeaway

A credible Binance USDⓈ-M perpetual backtest needs at least trade-price OHLCV candles, realized funding-rate events with their settlement timestamps, and mark-price data for unrealized PnL and liquidation checks. Treat these as separate time series in UTC and join by event time; the account's own funding ledger is a reconciliation source, not a substitute for public historical funding rates.

### Cited Findings

- Binance USDⓈ-M klines are uniquely identified by candle open time and return open/high/low/close, volume, close time and related fields; the endpoint accepts intervals and a maximum `limit` of 1,500, so longer history requires pagination. [Kline/Candlestick Data](https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/rest-api/market-data#kline-candlestick-data)
- Binance exposes a separate mark-price kline endpoint. Its OHLC records are useful for mark-to-market and liquidation checks, but they are not the same price series as trade-price klines. [Mark Price Kline/Candlestick Data](https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/rest-api/market-data#mark-price-kline-candlestick-data)
- The USDⓈ-M funding-history endpoint returns `fundingRate`, `fundingTime`, and the mark price associated with a funding fee charge. Its documented limit is 1,000 records per request. [Get Funding Rate History](https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/rest-api/market-data#get-funding-rate-history)
- Binance states that positive funding means longs pay shorts, funding is based on position nominal value and rate, and only positions held at the funding time incur the payment; the default interval is eight hours, but the venue may change an instrument's interval. [Introduction to Binance Futures Funding Rates](https://www.binance.com/en/support/faq/detail/360033525031); [Funding interval update notice](https://www.binance.com/en/support/announcement/detail/98d6b24d3e5c4f84a8ed04087997d8d0)
- `/fapi/v1/exchangeInfo` provides current exchange trading rules and per-symbol information, including contract type, status, onboard date, and symbol filters. It is a current snapshot; the docs do not describe a historical filters archive. [Exchange Information](https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/rest-api/market-data#exchange-information)
- Binance `Get Income History` is a signed `USER_DATA` endpoint and its income types include `FUNDING_FEE` and `COMMISSION`; it reports a particular account's realized ledger entries. This is distinct from the public market-wide funding-rate history. [Get Income History](https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/rest-api/account#get-income-history-user_data)
- Binance explains that liquidation is triggered by mark price reaching the liquidation price, while last price and mark price are distinct concepts. [Binance Futures Liquidation Protocols](https://www.binance.com/en/support/faq/detail/360033525271)

### Inferences

- Use separate normalized records keyed by `(venue, symbol, event_time, dataset_type)` for trade candles, mark candles, funding events and instrument metadata. Store UTC millisecond timestamps, source endpoint, retrieval time, raw payload hash and dataset version so a result can be reproduced.
- For a baseline indicator strategy, calculate indicators on trade-price candles. At a completed candle close, create the signal; the earliest ordinary market fill should be the next candle's open, with spread/slippage applied. Do not fill at the same close that was used to produce the signal. This is a causal design rule, not a Binance API guarantee.
- At each funding event, apply the realized historical rate to the position that is open at that event using the settlement mark price, with sign convention long pays on positive rate and short receives. Do not forward-fill a prior rate as if it were known or settled at later events. If a strategy itself consumes a funding-rate feature, historical realized funding is not necessarily a point-in-time record of when that rate became observable; use archived published/estimated snapshots or exclude that feature from MVP.
- Account funding payment history is appropriate later for reconciling a paper/live account's observed debits and credits against the simulator. It cannot provide a generic backtest history without private account access and cannot represent fills or funding for counterfactual positions.
- For initial data scope, choose liquid USDⓈ-M perpetual symbols and record the exchange metadata snapshot used. To avoid survivorship bias, a multi-symbol universe should account for delisted/inactive contracts and historical listing dates; current `exchangeInfo` alone cannot establish the full historical universe.
- Missing candles must be detected and reported, not silently synthesized by forward-filling OHLC. A gap changes indicator bar counts and makes intra-gap price path and stops unknowable. MVP policy: fail a run if the requested interval has material gaps, or explicitly segment the run and disclose the excluded windows. Keep raw data and gap diagnostics.

### Gaps

- Binance's cited public market-data endpoints document the current REST records but do not establish that every symbol/timeframe has complete historical coverage or an immutable historical contract-filter archive. Validate availability and gaps empirically when choosing the first symbol/timeframe.
- A funding settlement timestamp and associated settlement mark are documented, but the public history endpoint alone does not give every intermediate mark price or the exact sequence of market events around a settlement. Fine-grained event ordering needs archived finer-resolution data.

## 2. Execution, costs, leverage, margin and liquidation

### Takeaway

A futures backtest must report net PnL after both trading fees and funding, size positions against equity and contract rules, mark unrealized PnL with mark price, and represent liquidation/margin explicitly. Candle-only execution is an approximation: it cannot reveal queue position, exact spread, intrabar ordering or market impact.

### Cited Findings

- Binance says funding amounts are calculated from position nominal value and funding rate, with the settlement mark price tied to the funding charge; funding is exchanged between long and short holders rather than charged as an exchange trading commission. [Funding Rates](https://www.binance.com/en/support/faq/detail/360033525031); [Get Funding Rate History](https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/rest-api/market-data#get-funding-rate-history)
- Binance's current `exchangeInfo` provides symbol rules and filters; the endpoint describes itself as current exchange trading rules and symbol information. [Exchange Information](https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/rest-api/market-data#exchange-information)
- Binance's futures risk material explains initial and maintenance margin, leverage brackets, and that maintenance margin varies by notional tier. [Leverage and Margin of USDⓈ-M Futures](https://www.binance.com/en/support/faq/detail/360033162192); [Liquidation price calculation](https://www.binance.com/en/support/faq/detail/b3c689c1f50a44cabb3a84e663b81d93)
- Binance says liquidation occurs when mark price reaches the liquidation price and defines the maintenance-margin condition for triggering liquidation. [Binance Futures Liquidation Protocols](https://www.binance.com/en/support/faq/detail/360033525271)
- Binance documents order-book data as a market-data endpoint, which can support estimates of contemporaneous spread and displayed depth where archived snapshots are available. [Order Book](https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/rest-api/market-data#order-book)
- Binance's account income endpoint enumerates commission and funding-fee ledger types, supporting separate accounting buckets for these costs. [Get Income History](https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/rest-api/account#get-income-history-user_data)

### Inferences

- **Signal and fill rule:** evaluate only after a bar has closed; submit a simulated order for the next bar. If using stops/targets within a bar, use lower-timeframe or trade data to determine order. With OHLC-only ambiguity (both stop and target touched, or liquidation and stop touched), adopt the adverse outcome and mark the run as coarse-resolution.
- **Fee model:** charge a fee on every entry, exit and partial fill using notional × fee rate. If no account-specific historical rate is available, expose an explicit assumed taker rate for market orders, disclose it in the run config, and run sensitivity scenarios. Do not hard-code today's fee as though it were historical. Maker rebates/fees require limit-order fill assumptions and should be deferred.
- **Spread and slippage:** candle close/open is not a fill price. MVP can apply a disclosed, configurable per-side slippage in basis points on top of fees and show base and stressed-cost scenarios. A later execution model can derive spread/impact from archived order book or trades and scale impact with order size versus available depth. Do not claim candle-only fixed slippage predicts real fills.
- **Contract quantization:** size first from risk budget and entry/stop distance, then round quantity down to the symbol's step size and price to tick size; reject orders below minimum quantity/notional and recheck after rounding. Pull current filters at run time for current simulation and persist the version. Historical backtests need historical filters or must disclose use of current rules as a proxy.
- **Position sizing:** expose initial equity, target risk per trade, maximum notional/leverage, and isolated/cross margin choice. For MVP, prefer a single position in isolated-margin mode and conservative fixed leverage (or 1x) so portfolio cross-collateral interactions are not hidden. Leverage changes exposure and liquidation distance; it does not create edge.
- **P&L accounting:** maintain cash/equity, position quantity, entry VWAP, realized PnL, unrealized PnL at mark price, commissions, slippage and funding as separate fields. Produce both gross and net equity curves; net is the primary score. In linear USDⓈ-M, price PnL is quantity × price change, subject to venue contract semantics.
- **Liquidation model:** use mark-price intrabar high/low to detect if the liquidation threshold was crossed, and maintenance-margin tiers applicable to position notional. If candle resolution cannot establish whether a stop or liquidation happened first, assume liquidation/worse fill first and flag uncertainty. A simple static leverage multiplier is not a liquidation model.
- **MVP vs later fidelity:** MVP needs next-bar fills, explicit taker fee, conservative slippage, funding events, mark-price equity/liquidation checks, position sizing and filters. Defer queue-aware maker fills, full order-book replay, cross-margin portfolio liquidation, partial fills, insurance/ADL, latency, and venue-specific advanced order types until evidence/requirements justify their cost.
- **Data quality:** for a 1h baseline, align trade bar opens/closes, mark-price bars and funding event timestamps on UTC; maintain a separate timestamp grid rather than assuming every venue's settlement falls exactly on a bar edge. If event time falls inside a bar, use a finer mark/trade series or conservative event handling.

### Gaps

- The official source pages define exchange behavior and available fields, but the proposed fixed-bps slippage stress values and candle-level adverse ordering are engineering assumptions. Calibrate them from Trading Desk paper fills and archived quotes before treating them as realistic.
- Historical VIP/account fee rates and historical filter changes need an attributable archive. Public current endpoints are insufficient to reconstruct every user's past costs or all historical constraints.
- A faithful cross-margin liquidation simulation would require account-level collateral, other open positions, margin mode, bracket changes and exchange-specific risk logic; it is outside a one-strategy research MVP.

## 3. Strategy implementation and validation against look-ahead and overfitting

### Takeaway

Validation must preserve chronology and freeze a final holdout before tuning. A strategy that was repeatedly adjusted against the same history should be treated as selected from many trials; report the full trial count and use robustness statistics as diagnostics, not as a guarantee of future profitability.

### Cited Findings

- Bailey, Borwein, López de Prado and Zhu propose Probability of Backtest Overfitting (PBO) and combinatorially symmetric cross-validation (CSCV), and argue conventional holdout methods can be unreliable for investment backtests. [The Probability of Backtest Overfitting](https://escholarship.org/uc/item/4w1110bb)
- Bailey and López de Prado's Deflated Sharpe Ratio addresses selection bias from multiple testing and non-normal returns. [The Deflated Sharpe Ratio (author paper)](https://www.davidhbailey.com/dhbpapers/deflated-sharpe.pdf)
- Sullivan, Timmermann and White apply White's Reality Check bootstrap to a universe of technical trading rules to adjust for data-snooping bias across the rule universe. [Data-Snooping, Technical Trading Rule Performance, and the Bootstrap](https://www.fmg.ac.uk/publications/discussion-papers/data-snooping-technical-trading-rule-performance-and-bootstrap)
- Andrew Lo derives that multiplying a periodic Sharpe by the square root of periods is valid only under restrictive assumptions; serial correlation changes the correct annualization and statistical inference. [The Statistics of Sharpe Ratios](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=377260)
- The repository's `MACD Oscillator backtest.py` generates a position from moving averages computed on the same row's `Close`; it plots signals on that close and contains no execution, fee, funding, margin or liquidation simulation. The existing `Heikin-Ashi backtest.py` portfolio also calculates holdings/cash from same-row close and its stats annualize/compare in a way that is not an exchange futures ledger. These are code observations from local files, not claims about outside research.

### Inferences

- **Freeze the research question before tuning:** name one strategy hypothesis, instrument universe, timeframe, features, order timing, cost model, parameter ranges and evaluation metric. Register every tried parameter/model variant; do not report only the winner.
- **Causal feature contract:** at bar `t`, strategy code sees only records whose availability time is at or before the decision time. Indicators use trailing windows only; rolling calculations are right-aligned; normalization, volatility estimates and thresholds are fit on training windows only. Explicitly forbid centered rolling windows, future-filled features and revised labels leaking across the split.
- **Chronological split:** reserve the newest contiguous period as a locked final OOS set that is inspected once after design is frozen. Use earlier train/validation history for strategy selection. For nonstationary data, run expanding or rolling walk-forward folds and concatenate only each fold's unseen forward returns. Add purge/embargo around boundaries if holding periods or labels overlap the boundary.
- **Search-aware reporting:** preserve all variants tried, including failed experiments. If many variants are compared, report PBO/CSCV or a Deflated Sharpe diagnostic, and optionally a Reality Check/SPA-style test over the strategy family. These are not substitutes for a genuinely untouched holdout and correct data timing.
- **Required metrics:** net total return and annualized return; max drawdown and drawdown duration; return volatility; Sharpe with the sampling frequency, annualization method and serial-correlation caveat; downside risk/Sortino as supplementary; trade count, win rate, average win/loss, turnover/traded notional, exposure/time in market, long/short split, gross PnL, fees, slippage and funding contribution, worst trade/day, and liquidations/margin utilization. Plot net equity and drawdown. Keep flat periods in the return series.
- **Avoid metric traps:** do not rank only by Sharpe, CAGR or win rate. Include a no-trade benchmark and buy/hold reference where relevant, but avoid using an unrelated benchmark that changes the question. Sharpe from overlapping positions or smoothed returns needs serial-dependence-aware interpretation.
- **Robustness checks:** test neighboring parameter values rather than a single optimum; cost stress (e.g., higher slippage/fees), different market regimes, long/short separately, and reasonable start/end-date perturbations. Show whether profitability comes from a few extreme trades or a single symbol/time window.

### Gaps

- PBO/DSR/Reality Check methods require accurately recorded return series and the full candidate strategy set; no statistical correction can detect look-ahead that is built into the data pipeline or an unrecorded search process.
- There is no universally valid Sharpe, trade-count or return threshold that certifies a crypto-perpetual strategy. Acceptance depends on the intended strategy horizon, uncertainty, costs, capacity and operational context.

## 4. MVP shape and pragmatic acceptance criteria for a strategy from this repository

### Takeaway

Build the first service as a deterministic, research-only replay engine with a narrow strategy interface and a transparent Binance USDⓈ-M data adapter. The repository's MACD crossover is a plausible first port because its signal is simple and causal once shifted to next-bar execution, but its current script is an indicator/visualization prototype rather than an economically complete backtest.

### Cited Findings

- `MACD Oscillator backtest.py` computes short and long rolling averages on `Close`, enters long when short average is at least the long average and otherwise goes flat; it describes the classic moving-average trend idea and prompts for windows/dates/ticker. This is a direct repository inspection.
- The MACD file fetches Yahoo Finance data, plots signals at the same bar close, and does not model short exposure, exchange order constraints, commissions, funding or liquidation. This is a direct repository inspection.
- Binance's market-data API provides the relevant futures candle, funding-history, mark-price, exchange-info and order-book families needed to progressively supply a service. [USDⓈ-M Market Data API](https://developers.binance.com/en/docs/catalog/core-trading-derivatives-trading-usd-s-m-futures/api/rest-api/market-data)

### Inferences

- **Service boundary:** one internal research service accepts a versioned backtest request (strategy id/version/parameters, venue, symbol, interval, date range, initial equity, risk/cost settings) and returns immutable run id, data provenance, config hash, validation diagnostics, trades, net/gross equity curves, costs and metrics. Trading Desk should remain responsible for user/workspace authorization, job lifecycle and display; no order placement belongs in the research runner.
- **First strategy:** port `MACD Oscillator backtest.py` as a long/flat benchmark first; define an optional long/short variant as a separate strategy id, not an implicit reinterpretation of the source. Require `fast_window < slow_window`, warm-up bars, and a next-bar fill. Compare against no-trade and buy/hold baselines. Add other repository strategies only after each has a written rule specification and testable causal signal contract.
- **MVP dataset:** one or a small manually selected set of liquid `PERPETUAL` USDⓈ-M symbols; one fixed timeframe (suggest 1h for a practical first cut); trade OHLCV, mark-price bars, funding events, and a persisted exchange-info snapshot. Run data validation before simulation. Avoid broad altcoin-universe results until historical listings, delistings, filters and missing-data effects are understood.
- **MVP mechanics:** one-way net position, single active symbol/position, isolated margin, no pyramiding, market/taker orders, next-bar fill, explicit taker fee, conservative spread/slippage assumption, funding debit/credit at event, leverage/margin constraints, mark-price PnL and liquidation detection. Reports must state that OHLC execution and intrabar order are approximations.
- **Acceptance gates (engineering, not a profit promise):**
  1. A signal at bar close cannot fill before the next tradable timestamp; synthetic tests should prove no same-close look-ahead, including indicator warm-up and funding events.
  2. Re-running the same request against the same versioned data/config produces identical fills, equity and metrics; every run records code/version, data snapshot/hash and assumptions.
  3. Invalid symbols, non-perpetual instruments, unsorted/duplicate bars, material gaps, missing funding or mark data, and invalid contract increments fail with explicit diagnostics or a disclosed degraded mode—never silently fabricate values.
  4. Gross PnL reconciles to position changes and mark-to-market; net PnL reconciles gross less trading costs, funding and liquidation charges/penalties where represented. Long/short funding signs and quantity/tick rounding have deterministic accounting checks.
  5. Results display net as primary, gross separately, cost breakdown, turnover, drawdown, exposure, liquidation count and all parameters; they label assumptions/data gaps and avoid a single headline Sharpe as a verdict.
  6. Strategy review uses a locked chronological holdout plus forward walk-forward windows; candidate count is disclosed; there is no parameter search on the final holdout. Cost stress and adjacent-parameter robustness are included.
  7. Promote only to paper-forward observation after these integrity gates and a separately agreed OOS review; a profitable backtest alone is not a gate to live execution.
- **Phase sequence:** (A) write precise strategy and accounting spec; (B) normalize/cache/validate Binance data and metadata; (C) build deterministic event-driven replay for one strategy and single position; (D) add causal chronological validation and provenance; (E) paper-forward compare simulated vs observed fills/costs; (F) consider portfolio, advanced orders, other venues, or live execution as separate, explicitly approved product decisions.
- **Venue differences:** keep exchange-specific contract math, funding cadence/sign conventions, mark/index definitions, margin tiers, filters, fee tiers and liquidation logic behind adapters. A Binance USDⓈ-M implementation should not be presented as portable to COIN-M, Bybit, OKX, Hyperliquid or another venue without reconciling each venue's own contract and event rules.

### Gaps

- The repository does not currently supply a futures data adapter or a strategy interface with causal event-time semantics; the first port needs a written spec and independent accounting checks.
- Whether to start with only BTCUSDT or add ETHUSDT, exact timeframe, initial-equity/risk defaults and accepted holdout length are product decisions; this note intentionally does not set an investment allocation or promise performance.
- Paper-forward acceptance duration and allowable discrepancy between modeled and observed costs depend on Trading Desk's execution and data capture, which were outside this research assignment.
