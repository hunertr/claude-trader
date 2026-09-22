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
