# Research notes

I tested these rules through an MT5 Expert Advisor that trades the same entries as `pine/discrete price zones.pine` and adds exits, position sizing and risk controls. The EA stays private. Every figure below comes from the MT5 Strategy Tester on XAUUSD M15 with real-tick history and a 10,000 starting balance unless a line says otherwise.

## Pinning down the rules

My first written spec had two ambiguities that each changed every trade.

**Gap or pitch.** I described the zone spacing as 14.86. Read as a pitch, it puts the zone above the anchor at 3991.08. Read as the gap between zones, it gives a pitch of 17.50 and puts that zone at 3993.72. I overlaid both ladders on my hand-drawn 1H zones and the 17.50 pitch matched them. The script now takes the gap as input and derives the pitch.

**SMA or SMMA.** The spec said SMA 9. My chart ran SMMA 9, Wilder's smoothing, which lags much further. I coded the SMMA to match the chart I trade from.

## From a fixed target to a trailing exit

The first optimised configuration closed each trade one zone away, at 1.00 before the next zone line. It won often and made little money:

| Metric | Value |
|---|---|
| Trades | 409 |
| Win rate | 75.1% |
| Average win | +37.29 |
| Average loss | −93.66 |
| Profit factor | 1.19 |

The optimiser had widened the stop offset to 16.2 while the target stayed capped one zone away. An average loss 2.5 times the average win left the system one bad cluster from a losing year.

I replaced the fixed target with a trailing stop on the maximum favourable excursion (MFE). Once a trade gains a set amount, the stop follows the best price reached, minus a giveback, and never moves back. Six candidate parameter sets, optimised on 2024 to 2025 and run out of sample on January to July 2026, posted profit factors of 1.43 to 1.75 and win rates of 24% to 36%. The runner gives back some trades a fixed target would have banked, which lowers the win rate, and lets the larger winners run far enough to pay for it.

## In-sample versus out-of-sample

One set, run 8, stood out:

| Metric | In-sample (2024 to 2025) | Out-of-sample (Jan to Jul 2026) |
|---|---|---|
| Profit factor | 1.66 | 1.96 |
| Trades | | 81 |
| Win rate | | 43.2% |
| Average win / loss | | +252.57 / −97.71 |
| Max equity drawdown | | 7.37% |

Run 8 began from a 9,528 balance. The out-of-sample profit factor beat the in-sample figure. Curve-fitted parameter sets tend to do the reverse, so I read this as the best evidence of robustness in the project. With 81 trades, one bad month could move these numbers a long way, so I treat them as a lead, not a verdict.

A later full-period run, January 2024 to July 2026:

| Metric | Value |
|---|---|
| Trades | 261 |
| Net profit | +5,309.69 |
| Profit factor | 1.71 |
| Win rate | 35.6% |
| Average win / loss | +137.14 / −44.20 |
| Max equity drawdown | 5.82% |
| Recovery factor | 8.69 |
| Equity curve LR correlation | 0.96 |

## The 2021 to 2023 losses

I extended a later multi-position build back to January 2021. The balance fell from 10,000 to about 5,300 by August 2023, then climbed to about 20,000 by mid-2026. I read both figures off the equity chart.

The entries looked right on the chart through the whole period. The exits failed. The trail gave back a fixed 35.4 points from the peak. The initial stop sat about 13.84 below an entry on the zone line, so the trailed stop only rose above it once a trade gained 21.56 points. Any trade that peaked below that fell back to the full initial loss.

Gold traded near 1,800 to 2,000 in 2021 to 2023 and near 4,000 in 2026. The same 35.4 points equals 1.8% to 2.0% of price in the first period and 0.9% in the second. A move of the same percentage size covered half as many points in 2021 to 2023, so fewer trades reached the level where the trail engaged.

I replaced the fixed giveback with a percentage of the MFE: the stop sits at peak minus p% of the profit reached. It engages from the first point of profit and scales with the size of the move. In a simulation of a trade that peaked 20 points up and then reversed, the fixed trail returned −13.84 and a 40% giveback returned +12.00. I have not published the backtest of this change yet.

The zone pitch has the same scale problem. 17.50 points equals 0.97% of price at 1,800 and 0.44% at 4,000. I have not tested a pitch that scales with price.

## Ideas that failed

| Change | Result |
|---|---|
| Drop the 1H and 4H trend filter and take direction from the M15 chart | Failed in testing. I restored the filter. |
| Require the dollar index (DXY) to trend against gold on M15, 1H and 4H | Broke the strategy. I removed it. |
| Block trades while ADX(14) sits at or below 22.65 or falls | Did not fix the 2021 to 2023 losses, which came from exits. I removed it. |
| Port the rules to US100 with its own anchor (22327.25 / 22374.60), gap 243.73 and SMMA 7 | Failed. The gold version stayed the focus. |

## Rule 5 does nothing at the current defaults

The zone formula keeps Close − Uₙ below the pitch of 17.50 (proof in [methodology.md](methodology.md)). The optimiser searched the late-entry maximum between 5 and 50 and chose 49.5, so the check never blocks a trade. The live system trades on four rules. To bring rule 5 back, set the maximum below 17.50.

## Timeframes

All figures above come from M15. Backtests of the 1H and 2H signals came out profitable as a swing strategy, and those are the timeframes I run the alerts on. Every distance parameter carries M15 scale, so treat each new timeframe as a fresh optimisation.

## Bugs I caught

AI wrote most of the code in this project. Each of these bugs compiled and passed the syntax checks before I found it:

- **Zero balance at attach.** The EA read the account balance before the terminal had synced, got 0, and set the daily loss limit to 3% of 0. The breaker could never trip, and the dashboard read ARMED. The fix waits for a non-zero balance and blocks trading until the breakers hold valid values.
- **Breaker reset on recompile.** Recompiling after a −3% day reset the day's starting balance to 9,700 and re-armed the breaker. It then allowed another 3%, a 5.91% day against a 5% prop-firm limit. The fix stores breaker state in terminal global variables so it survives a recompile or restart.
- **One trailing state for many positions.** Three global variables held the MFE peak for a single position. With two trades open, they overwrote each other's peaks. The fix keeps a peak per ticket.
- **Dashboard wiping to zero.** The performance panel zeroed its stats at the start of each refresh, and returned early when the trade-history query failed. Any transient failure blanked the panel. It now keeps the last good result.
- **Pine helpers inside a conditional.** The first dashboard version declared its helper functions inside an `if` block, which Pine rejects. I caught it in review before publishing.
