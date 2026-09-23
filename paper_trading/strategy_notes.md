# Trading Strategy Notes

Reference doc for both the paper portfolio and the live "Agentic" Robinhood
account once funded. Read by the hourly automated check-in before making
decisions, so strategy stays consistent across runs instead of drifting.

## Backtest results (2026-09-22) — read this before trusting the rules below

Ran a real historical backtest (Robinhood daily OHLC, Jan 2025–Sep 2026) of
the literal entry rule below in isolation: SMA50 trend filter + RSI(14)
crosses above 58 + MACD confirms, long-only, fixed 5% stop / 10% target,
one position at a time. Results:

- **NVDA**: 11 trades, 27% win rate, **-4.3% return** vs **+60.7% buy-and-hold**
  over the same period. Profit factor 0.73 (net losing).
- **SPY**: 4 closed trades, 50% win rate, +3.9% return vs +30.3%
  buy-and-hold. Profit factor ~2, but n=4 is not statistically meaningful.

**Verdict: this exact rule set, tested in isolation, loses to just holding
the asset.** Root cause: RSI>58 fires after a move is already extended, so
entries are late; the fixed 5% stop then gets hit on normal pullbacks
before the larger continuation move plays out (8 of 11 NVDA trades were
stop-outs, several right before the stock kept running). This is the same
failure mode this repo's MT5 bot already diagnosed and fixed for
EURUSD/GBPUSD via SMC confluence (see backtest results below and
CLAUDE.md) — pure RSI-momentum entries without structural confirmation
enter too late.

**Implication for automated runs**: do not treat the raw RSI/MACD/SMA
entry rule as sufficient on its own for real-money sizing decisions. The
TJR/ICT sweep→BOS→reaction sequencing (below) is the more promising fix —
it deliberately waits for the pullback/reaction instead of chasing the
RSI cross — but it has NOT been backtested yet. Prefer entries that
satisfy the ICT sequencing over a bare RSI cross when the two disagree.
Trailing the stop instead of a fixed 5% target/stop (letting winners run
further, consistent with the "trend-following has the deepest edge"
research below) is also a more promising fix than tested here, and is
already how the live MT5 bot's MONITOR/WATCH agents behave — the same
should apply to the crypto account once funded rather than a bare
fixed-% exit. Full methodology, limitations (daily bars only, no
fees/slippage, small sample, SMA50 as EMA200 proxy), and trade log
available in this repo's commit history / session log if needed.

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

**Sizing formula fix (found via backtest, 2026-09-22):** risk_amount /
stop_distance_pct alone is NOT a valid position-size formula on its own —
with a 2% risk cap and a 5% stop, it computes to 40% of capital in a
single trade, which blows through the 20% max-deployed rule above before
even considering a second position. The formula must be:

```
position_notional = min(risk_pct_of_capital / stop_distance_pct, max_single_trade_pct) * capital
```

Use `max_single_trade_pct = 10%` (leaves room for up to 2 concurrent
positions under the 20% total-deployed cap). Whichever of the risk-based
size or the 10% cap is SMALLER wins — the risk-based number is not an
entitlement to trade that large just because the stop happens to be
tight. Apply this on every position-sizing decision, paper or live.

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

## Weekend liquidity risk (crypto-specific, relevant every Fri-Sun)

Research confirms crypto trading volume drops 20-40% on weekends vs
weekdays, with some data showing 2-3x weekday volatility for BTC/alts
during that window -- driven by thin order books, not new information.
One documented extreme: $19B in long positions liquidated following a
Friday close as volatility spiked into the low-liquidity weekend window.

Practical implications, not just background:
- A resting stop-loss can suffer worse slippage than modeled if it
  triggers during weekend thin liquidity -- a fast wick can blow through
  the stop price before it fills. This is a real execution-quality risk
  on any position held into a weekend, not a reason to widen stops
  (that's loosening risk management, not managing this one).
- Be more conservative about opening brand-new positions right as
  weekend liquidity thins (Friday evening UTC onward) -- wider spreads
  eat more of any edge during that window specifically.
- Do not treat "crypto trades 24/7" as "crypto behaves the same 24/7" --
  weekend price action reflects thin markets more than trend conviction.

## Crypto universe — what's actually tradable and worth looking at

Checked Robinhood's full crypto catalog (2026-09-22). Most of it is
low-liquidity, high-volatility meme/micro-cap tokens, many with **active
trading halts** (BILL, CASHCAT, MOODENG, PNUT, MEW, POPCAT, FLOKI, PENGU,
TRUMP, WIF, and more — several halted in NY at the time of checking).
BONK, already sitting in the real account, is in this same category. Do
not treat "look for faster profit" as license to trade this part of the
catalog — it's the highest-risk, lowest-quality segment, not an
opportunity set.

**Preferred watchlist beyond BTC/ETH/SOL** — established, liquid,
not halted at last check: SUI, SEI, ARB, OP, INJ, AAVE, RENDER, HBAR.
Snapshot check (2026-09-22) showed no real setup on any of them — all
moving under ±2.5% on the day, no breakout or trend signal. That's a
legitimate "nothing to do right now," not a reason to widen the search
into the meme-coin tier instead.

**Known data limitation**: Robinhood's tools give live crypto quotes but
NOT historical OHLC bars or technical indicators for crypto (unlike
equities, which have full RSI/MACD/SMA history). This means crypto
entries can't be validated with the same rigor as the NVDA/SPY backtest
— only live price and 24h change are available. Until a better crypto
data source exists, lean more conservative and patient on crypto entries
than the equity-validated rules would suggest, and don't mistake "I can't
verify this technically" for "so any move is as good as another."

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

**Order mechanics confirmed via $100 backtest rerun (2026-09-22):** every
trade at this size requires fractional units. For equities, fractional
shares may not support a broker-side resting stop-loss order (backtest
had to assume clean fills, which live equity trading can't guarantee).
For crypto specifically — what the live account actually trades — this
is NOT a blocker: `place_crypto_order` supports `stop_loss` and
`stop_limit` order types natively on fractional quantities. Always use a
real resting stop order for live crypto trades rather than only a
mentally-tracked stop level; do not rely on the hourly job polling price
and firing a market sell as the primary stop mechanism — that adds
avoidable slippage/latency risk a resting stop order doesn't have.

## Free course study — trading plan & behavioral finance

Studied two well-regarded free courses for content not already covered
above: BabyPips "School of Pipsology" (the most established free forex
curriculum — 350+ lessons, free, self-paced) and Yale's "Financial
Markets" (Robert Shiller, Nobel laureate, free on Coursera). Two genuinely
new additions, not just repetition of the quant research above:

**A written trading plan is what separates trading from gambling.**
BabyPips' framing, and it's a fair one: before scaling risk, the plan
should state — in writing — goals, risk tolerance, time horizon, and which
setups are/aren't traded. This repo already has the equivalent (this
strategy doc + the risk rules + the explicit "no options, crypto only,
1-2% per trade" constraints for the $40 account), so the gap isn't a
missing plan, it's discipline in *following* the existing one under
pressure to "prove" faster results — worth naming explicitly since that
pressure is real (see: repeated pushes to loosen risk for bigger returns).

**Markets are not fully efficient — behavioral finance explains *why*
trend-following works, not just *that* it works.** Shiller's course
(built on his own Nobel-winning research) argues persistent market
anomalies come from narrative-driven feedback loops and overconfidence —
including among professionals, not just retail. This reframes the
trend-following edge documented above: it isn't a statistical fluke that
happens to persist, it's a structural consequence of how humans actually
process and react to market narratives (slow to react to new information,
then overreact and chase). Practical takeaway: overconfidence is the
single most consistently cited failure mode in both the quant and
behavioral-finance literature reviewed so far — a good reason to keep
applying the existing risk caps mechanically rather than trusting
in-the-moment conviction on any one trade or "hot streak."

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
- https://www.babypips.com/learn/forex (School of Pipsology, free course)
- https://www.coursera.org/learn/financial-markets-global (Shiller, Financial Markets, free)
- https://oyc.yale.edu/economics/econ-252
