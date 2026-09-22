# Trading Strategy Notes

Reference doc for both the paper portfolio and the live "Agentic" Robinhood
account once funded. Read by the hourly automated check-in before making
decisions, so strategy stays consistent across runs instead of drifting.

## Indicator set (crypto + equities)

Research-backed combination, not stacking redundant signals:
- **Trend**: EMA (fast/slow cross) or price vs 50/200 SMA
- **Momentum**: RSI(14) — >58 bullish conviction, <42 bearish, 42-58 ambiguous
- **Confirmation**: MACD — trade only when MACD agrees with RSI direction
- **Volatility/entry timing**: Bollinger Bands — squeeze = breakout pending,
  price at band extreme + RSI divergence = mean-reversion entry
- Volume as a confirmation layer, not a standalone signal

Avoid stacking indicators that measure the same thing (e.g. RSI + Stochastic
together add noise, not signal).

## Risk management (hard rules, not suggestions)

- Max 1-2% of account risked per trade (position size, not just notional)
- Max 20% of account deployed at once
- Daily loss cap: 3% — halt new entries for the rest of the day if hit
- Every position gets a stop-loss and take-profit set at entry, no exceptions
- Never add to a losing position
- No options trading

## Entry sequencing (TJR / ICT liquidity model)

This repo's MT5 bot already detects raw SMC building blocks per-instrument
(`_smc_detect_fvg`, `_smc_detect_ob`, `_smc_detect_liquidity` in gold.py /
eurusd.py / gbpusd.py). TJR's refinement is the *sequencing* those signals
fire in — don't trade an FVG/OB in isolation, require this order:

1. **Identify liquidity** — an obvious swing high/low where stops cluster
   (prior session high/low, equal highs/lows, round numbers)
2. **Wait for the sweep** — a wick through that level that closes back
   inside (a stop hunt, not a genuine breakout)
3. **Confirm break of structure (BOS)** — after the sweep, price must break
   the most recent opposing swing point on a lower timeframe, confirming
   the reversal direction
4. **Enter on the reaction** — at the resulting FVG or order block formed by
   the BOS impulse, not on the sweep candle itself
5. **Stop beyond the sweep's extreme**, target the opposing liquidity pool

Applies to session-based levels too (e.g. Asia-range high/low swept at
London open, then BOS confirms direction) — same pattern, different level
source. For crypto (24/7, no sessions in the FX sense), substitute prior
day/week high-low or equal highs/lows as the liquidity reference instead of
session ranges.

Discipline addition from TJR's own framing: cap trades per day/week (quality
over quantity) rather than trading every signal that appears — this is
consistent with, not in tension with, the 1-2% per-trade risk rule above.

## Broader research synthesis

Wider pass across quant/systematic trading literature, not just retail
day-trading content, to sanity-check the approach above.

**Trend-following has the deepest documented edge.** AQR and others find
trend/momentum effects persistent across 200+ years and multiple asset
classes; systematic trend-following (the Turtle Trading lineage, $300B+ in
CTA assets today) was positive in 8 of the last 10 major market crises.
Practical implication: default bias is *trade with the prevailing trend*
(the EMA200 gate this repo's MT5 bot already enforces is the right
instinct) — treat counter-trend/mean-reversion entries as the higher-bar,
lower-frequency exception, not the default mode.

**Mean reversion is real but shorter-horizon and asset-dependent.** Academic
work (Lehmann) finds ~1-2%/week abnormal returns after extreme short-term
moves, but it works better in stocks than in trending assets like crypto,
which has favored momentum historically. Reserve BB-extreme mean-reversion
entries for range-bound conditions (low ADX / weak trend), not as a
default crypto approach.

**Base rate on day trading is bad, and it's frequency-driven, not
strategy-driven.** Across studies, 70-95% of day traders lose money;
losing traders place roughly 4x more trades than winning ones, and
accounts trading 500+ times/year show up to an 80% loss rate. This
directly reinforces the "cap trades per day/week, quality over quantity"
rule already in this doc — it is not just TJR's opinion, it's the
single most consistent finding across the retail-trading literature.
Overtrading is the most likely failure mode here, more than picking a
wrong indicator.

**Position sizing: fractional Kelly, not flat guesses.** The Kelly
criterion sizes bets from actual win rate and win/loss ratio, but with
under ~50 realized trades those estimates carry too much variance to
trust directly — a 20-trade sample can be off by 10+ points on win rate
alone. Practical rule: keep using the flat 1-2% cap (equivalent to a
conservative quarter-Kelly for a plausible retail edge) until the
trade_log in portfolio.json has 50+ closed trades, then actually compute
realized win rate and average win/loss ratio from that log and revisit
sizing with real numbers instead of an assumed edge.

**Overfitting is the main way a backtested "edge" turns out fake.**
Over 90% of academic/backtested strategies reportedly fail when traded
with real capital — mainly from in-sample parameter tweaking and
lookahead bias. Guardrails already in place that address this: the
indicator set is deliberately small and non-redundant (not curve-fit to
this account's short history), and parameters (RSI 58/42, EMA200, etc.)
come from the existing MT5 bot's independently-run 2-year backtest, not
from tuning against this paper portfolio's own results. Do not start
adjusting thresholds specifically to make recent paper/live trades look
better in hindsight — that's overfitting to a live sample of one.

## Small-account reality (applies directly to the $40 live account)

Research confirms: accounts under ~$1,000 carry real risk of ruin from fee
drag and position-sizing constraints — the fixed cost per trade eats a much
bigger share of a small account than a large one. Implications for the $40
account specifically:
- Favor fewer, higher-conviction trades over frequent scalping — fees will
  dominate returns on a $40 base if traded like a $10k account
- Crypto only (24/7 market, no PDT restriction — unlike equities/margin)
- Position sizing at this scale will necessarily be a much larger % per
  trade than the 1-2% rule implies (there's no way to diversify $40
  meaningfully) — treat this as elevated real risk per trade, not a reason
  to abandon stops/targets
- Do not chase multiplying $40 into a large sum quickly — that expectation
  is not realistic from legitimate trading edge; the point is to run the
  same disciplined process that scales, and let compounding work over time

## Sources consulted (2026)

- https://www.kucoin.com/blog/day-trading-crypto
- https://coinbureau.com/guides/risk-management-strategies-crypto-trading
- https://www.mexc.com/news/549058
- https://www.theblockverse.co/best-crypto-indicators-for-beginners/
- https://tokenmetrics.com/blog/10-best-indicators-for-crypto-trading-and-analysis-in-2026/
- https://www.snappchart.app/blog/beginner-playbook/tjr-ict-trading-strategy
- https://phidiaspropfirm.com/trading-strategies-explained
- https://www.researchgate.net/publication/369427448_Comparison_of_Two_Quantitative_Strategies_Momentum_and_Mean-reversion
- https://papertradingjournal.com/2026/05/12/momentum-vs-mean-reversion-statistics/
- https://medium.com/@faisal_haroon/i-reviewed-every-major-day-trading-study-from-the-last-25-years-the-data-is-devastating-4b116273b956
- https://tradeciety.com/24-statistics-why-most-traders-lose-money
- https://darkbot.io/blog/kelly-criterion-crypto
- https://www.altrady.com/blog/risk-management/kelly-criterion-crypto-position-sizing
- https://en.wikipedia.org/wiki/Trend_following
- https://www.luxalgo.com/blog/what-is-overfitting-in-trading-strategies/
- https://blog.quantinsti.com/walk-forward-optimization-introduction/
