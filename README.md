# Discrete Price Zones

A Pine Script v6 indicator for XAUUSD. It builds a ladder of support and resistance zones from one anchor zone and a fixed pitch, then marks a long or short entry when five rules pass on a closed bar.

---

AI has changed how I build automated retail trading strategies.

Writing the code takes a fraction of the time it used to. The harder part is the math: thinking in probabilities, deciding what you can quantify, and turning a trading idea into a discrete, ordered set of rules.

Take support and resistance. You can define the zones with arithmetic and stop drawing them by hand. Pick one anchor zone with lower boundary L and upper boundary U, plus a pitch D, the distance from one zone's lower line to the next zone's lower line.

Every zone follows from

$$\text{Zone}_n = [L + nD,\ U + nD]$$

where n = 0 is the anchor zone, n = 1 is the zone above it, n = −1 is the zone below it, and so on in both directions.

The close alone tells you which zone sits nearest below price, with no scan through past bars:

$$n = \left\lfloor \frac{\text{Close} - U}{D} \right\rfloor$$

A long entry then reduces to three checks on the signal candle:

1. The candle closes above the zone: Close > Uₙ
2. The candle's wick retests the zone: Low ≤ Uₙ + tolerance
3. Price has not run too far past it: Close − Uₙ < maximum distance

Add a higher-timeframe trend filter and a filter that blocks choppy regimes, and you have the skeleton of a systematic strategy before you write any code.

The hard work sits upstream of the code: deciding what to build, writing rules that leave no room for interpretation, testing whether the idea has an edge, and understanding why the strategy behaves the way it does.

You also have to understand the code. AI writes a strategy in minutes, and it can slip in bugs that pass every syntax check. In the MT5 version of this system, an AI-written circuit breaker read the account balance before the terminal had synced with the broker. It got zero, computed a daily loss limit of 3% of zero, and could never trip. The dashboard still read ARMED. If an algorithm opens a trade it shouldn't have, you're the one who reads the logs, traces the logic, finds the failure and fixes it.

You learn something from each strategy you build: which assumptions you can quantify, which ideas survive translation into rules, and which ones fall apart once you test them. [docs/research notes.md](docs/research%20notes.md) logs what failed on this one.

All of this applies to retail systematic strategies. Institutional desks face harder problems in data, execution, infrastructure, research methodology and risk.

AI speeds up implementation. You still have to understand the system you're building.

---

## What the indicator draws

| Element | Detail |
|---|---|
| Zone ladder | 121 boxes: the zone nearest price plus 60 above and 60 below, extended across the full chart width. The boxes sit at fixed prices, so they do not move with price or repaint. |
| SMMA | 9-period smoothed moving average (`ta.rma`) on the chart timeframe. |
| Signals | BUY triangles under the bar, SELL triangles above it, and a faint background tint, on bars where all five rules pass. |
| Market-state table | Trend per timeframe, the chop filter, the active zone, distances to the zone and SMMA, a five-item checklist and a status line. |
| Entry alerts | One push notification per signal with entry, zone, stop and a lot size, locked to one chart timeframe. |

`pine/chop filter.pine` is a companion script that plots the Choppiness Index in its own pane. It colours the line with the same rule the entry uses, so the line turns red on the same bars the filter blocks.

## The zones

The defaults reproduce the zones I drew by hand on the 1H chart:

| Symbol | Value | Meaning |
|---|---|---|
| L | 3976.22 | Anchor zone lower line |
| U | 3978.86 | Anchor zone upper line |
| h = U − L | 2.64 | Zone height |
| g | 14.86 | Gap from one zone's upper line to the next zone's lower line |
| D = g + h | 17.50 | Pitch between matching lines of neighbouring zones |

The script takes `g` as its input and computes `D = g + h`. With the defaults, zone 3 spans 4028.72 to 4031.36 and zone 14 spans 4221.22 to 4223.86, which you can see as the active zone in the table above.

For shorts, the nearest zone above price comes from the ceiling:

$$n = \left\lceil \frac{\text{Close} - L}{D} \right\rceil$$

## Entry rules

A long fires on a closed bar when all five pass. Shorts mirror each rule.

| # | Rule | Long condition (defaults) |
|---|---|---|
| 1 | Trend | Close > SMMA₉ on 1H **and** Close > SMMA₉ on 4H |
| 2 | Regime | CHOP₁₄ < 50.5 **and** CHOP₁₄ ≤ its value one bar earlier |
| 3 | SMMA hold | Previous close > previous SMMA, Low ≤ SMMA + 1.56, Close > SMMA |
| 4 | Zone retest | Close > Uₙ **and** Low ≤ Uₙ + 0.55 |
| 5 | Distance | Close − Uₙ < 49.5 |

Rule 5 never blocks a trade at these defaults. The floor in the zone formula keeps Close − Uₙ between 0 and D = 17.50, so any maximum above 17.50 lets every entry through. The optimiser settled on 49.5, which switches the check off. [docs/methodology.md](docs/methodology.md) has the proof, worked examples for both directions, and the formula for each indicator.

## Default parameters

The defaults match the parameter set my MT5 Expert Advisor trades live, tuned with MT5's genetic optimiser on XAUUSD M15. All distances are in price units, so 1.00 means one US dollar on gold.

| Input | Default | Role |
|---|---|---|
| Anchor lower line | 3976.22 | L |
| Anchor upper line | 3978.86 | U |
| Gap between zones | 14.86 | g, gives D = 17.50 |
| Zones drawn each way | 60 | Ladder size above and below price |
| SMMA period | 9 | Chart timeframe plus the 1H and 4H trend filter |
| CHOP period | 14 | Choppiness Index lookback |
| CHOP block if ≥ | 50.5 | Regime threshold |
| Zone retest tolerance | 0.55 | Rule 4 wick allowance |
| SMMA retest tolerance | 1.56 | Rule 3 wick allowance |
| Late-entry max | 49.5 | Rule 5, inactive while above D |

## Market-state table

| Row | Meaning |
|---|---|
| 15m / 1H / 4H | BULL if close is above that timeframe's SMMA, BEAR if below. The 15m row reads the chart timeframe, whatever it is. |
| Verdict | LONGS ONLY or SHORTS ONLY when 1H and 4H agree, MIXED otherwise. MIXED blocks every entry. |
| CHOP | Current value with an arrow for its direction. Red at or above 50.5, or while rising. |
| Active | Lower and upper line of the zone the current bias points at. |
| Px vs line | Signed distance from close to the zone line rule 4 tests. |
| Px vs SMMA | Signed distance from close to the SMMA. |
| Checklist | ✓ or ✗ for rules 1 to 5 in the bias direction, and a count out of 5. |
| Status | BUY FORMING, SELL FORMING, NO-TRADE (chop), NO BIAS (mixed), or WAITING with the number of rules still failing. |

## Alerts

The alert module reads the same five conditions as the markers. It skips the display toggle, so hiding the markers leaves alerts running. Each alert fires once, when the bar closes, and its message carries the trade. Messages start with DPZ, short for Discrete Price Zones:

```
DPZ SELL | XAUUSD 2H
Entry ~4220.40 (bar close)
Zone 4221.22 / 4223.86
SL 4235.06  (14.7 pts)
Lots 0.06 @ 1% of 10000
Exit: trail the move, no fixed TP
```

To set it up:

1. Open the chart on the timeframe you want alerts from, for example 2H.
2. In the indicator's **Entry Notifications** settings, set *Notify only on this timeframe* to match. Fill in account size, risk % and your broker's contract size. The badge in the bottom right should read `ALERTS ARMED 2H`.
3. Create an alert with condition **Discrete Price Zones → Any alert() function call**. Skip the older `DPZ BUY` and `DPZ SELL` entries, which have no timeframe lock or trade details.
4. Tick *Notify in app* and sign in to the same account on your phone to get push notifications.

An alert keeps running the script version that existed when you created it. After you edit the script, delete the alert and create it again.

## Installing

1. Open the Pine Editor.
2. Paste `pine/discrete price zones.pine`, save, and add it to an XAUUSD chart.
3. Add `pine/chop filter.pine` as well if you want the oscillator pane.

## Limitations

- **Entries only.** The indicator marks where a trade would open. The exits, a trailing stop on the maximum favourable excursion, live in the MT5 EA and are not in this repository.
- **Close-based signals.** A marker on the bar still forming can appear and vanish until that bar closes. Judge closed bars only. Alerts wait for the close.
- **Higher-timeframe values on live bars.** The script requests the 1H and 4H SMMA with `lookahead_off`. On historical bars the 4H value updates when each 4H bar completes. On the live bar it follows the 4H bar still forming. A 2H bar that closes halfway through a 4H bar can see a different 4H SMMA live than it did in history. The gap only matters when price sits on the 4H SMMA.
- **Timeframe semantics.** On a 2H chart the "1H" filter requests a lower timeframe than the chart and returns the last 1H value inside each 2H bar. The CHOP filter and the table's 15m row both run on the chart timeframe.
- **Instrument-specific geometry.** The zones sit at absolute gold prices. On another instrument, or on gold at a very different price level, you need your own anchor and pitch.
- **Parameters tuned on M15 in MT5.** Tolerances and thresholds came from optimising the EA on M15 bars. Re-optimise before you rely on them for another timeframe.
- **No Pine backtest.** The script uses `indicator()`, so Pine's strategy tester has nothing to run. The results in the research notes come from the MT5 Strategy Tester on real ticks.
- **15m label.** The script shows a red "Parameters tuned on the 15m chart" label on other timeframes. The label predates the 1H and 2H alert work and has no effect on signals.

## Repository layout

```
.
├── README.md
├── LICENSE
├── pine/
│   ├── discrete price zones.pine   zones, SMMA, signals, table, alerts
│   └── chop filter.pine            Choppiness Index pane
└── docs/
    ├── methodology.md        formal rules, proofs, worked examples
    └── research notes.md     backtest numbers, failed ideas, bugs
```

## License

MIT. See [LICENSE](LICENSE).

## Disclaimer

Research and educational code, not financial advice. Backtests simulate fills, and your broker's spread, commission and slippage will differ. Trade at your own risk.

Built by Thabo Claude Chipokolo.
