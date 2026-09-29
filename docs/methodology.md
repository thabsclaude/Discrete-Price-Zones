# Methodology

This document specifies every rule in `pine/discrete price zones.pine` in enough detail to reimplement it in another language. All prices are in US dollars per ounce and all distances in price units.

## Notation

| Symbol | Meaning | Default |
|---|---|---|
| L, U | Anchor zone lower and upper line | 3976.22, 3978.86 |
| h = U − L | Zone height | 2.64 |
| g | Gap from a zone's upper line to the next zone's lower line (script input) | 14.86 |
| D = g + h | Pitch | 17.50 |
| τ_z | Zone retest tolerance | 0.55 |
| τ_m | SMMA retest tolerance | 1.56 |
| δ | Late-entry maximum | 49.5 |
| N | SMMA period | 9 |
| k | CHOP period | 14 |
| θ | CHOP threshold | 50.5 |

Subscript t marks the signal bar, the most recent closed bar. C, H and Lo stand for its close, high and low.

## 1. Zone geometry

Zone n covers

$$Z_n = [L_n, U_n] = [L + nD,\ U + nD], \quad n \in \mathbb{Z}$$

The script takes the gap g as input because I measured g on the chart. D follows as g + h. Mixing the two up shifts every zone after the anchor: with D = 14.86 instead of 17.50, zone 3 would start at 4020.80 instead of 4028.72.

| n | L_n | U_n |
|---|---|---|
| −1 | 3958.72 | 3961.36 |
| 0 | 3976.22 | 3978.86 |
| 1 | 3993.72 | 3996.36 |
| 3 | 4028.72 | 4031.36 |
| 4 | 4046.22 | 4048.86 |
| 14 | 4221.22 | 4223.86 |

The script draws zones n* − 60 to n* + 60, with n* = round((C − L) / D), as boxes extended across the chart. The boxes sit at fixed prices, so zooming or scrolling never moves them.

## 2. Selecting the zone under test

A long tests the highest zone whose upper line sits at or below the close:

$$n_{\text{long}} = \left\lfloor \frac{C - U}{D} \right\rfloor$$

A short tests the lowest zone whose lower line sits at or above the close:

$$n_{\text{short}} = \left\lceil \frac{C - L}{D} \right\rceil$$

Both take one division and one rounding per bar. No loop over past bars runs.

**Bound on the distance.** From the floor, (C − U)/D − 1 < n ≤ (C − U)/D. Multiply by D and rearrange:

$$0 \le C - U_n < D$$

The ceiling gives the mirror result for shorts, 0 ≤ L_n − C < D.

**Consequence for rule 5.** Rule 5 blocks a long when C − U_n ≥ δ. The bound caps that distance below D = 17.50, so any δ > 17.50 lets every entry through. The optimised δ = 49.5 therefore disables rule 5, and the live system runs on rules 1 to 4. To make rule 5 do work, pick δ below 17.50. The spec I started from used δ = 5.

## 3. Trend filter

$$\text{Long allowed} \iff C_t > \text{SMMA}_N^{1H} \ \land\ C_t > \text{SMMA}_N^{4H}$$

$$\text{Short allowed} \iff C_t < \text{SMMA}_N^{1H} \ \land\ C_t < \text{SMMA}_N^{4H}$$

The filter compares the chart's close against the higher-timeframe averages. It does not compare the 1H close with the 1H average. Disagreement between 1H and 4H blocks both directions.

The smoothed moving average uses Wilder's recursion, which Pine exposes as `ta.rma`:

$$\text{SMMA}_t = \frac{(N - 1)\,\text{SMMA}_{t-1} + C_t}{N}$$

`ta.rma` seeds the recursion with the simple average of the first N closes. With N = 9 each new close carries a weight of 1/9, so the SMMA lags further than a 9-period SMA.

The script requests both averages with `request.security(..., lookahead=barmerge.lookahead_off)`. Section 8 covers what that means on live bars.

## 4. Regime filter

The Choppiness Index over k bars:

$$\text{CHOP}_t = 100 \cdot \frac{\log_{10}\!\left( \sum_{i=0}^{k-1} \text{TR}_{t-i} \,\big/\, (\max H_{t-k+1..t} - \min Lo_{t-k+1..t}) \right)}{\log_{10} k}$$

TR is the true range. A value near 100 means price covered a lot of distance bar to bar but went nowhere. A value near 0 means price travelled in a straight line.

$$\text{Regime OK} \iff \text{CHOP}_t < \theta \ \land\ \text{CHOP}_t \le \text{CHOP}_{t-1}$$

The second clause blocks entries while chop is rising, even below θ. CHOP runs on the chart timeframe.

## 5. SMMA hold

For a long:

$$C_{t-1} > \text{SMMA}_{t-1} \ \land\ Lo_t \le \text{SMMA}_t + \tau_m \ \land\ C_t > \text{SMMA}_t$$

The previous bar closed above the average, the signal bar's low came within τ_m of it, and the signal bar closed above it. The rule looks for a pullback to the SMMA inside an existing move above it. A bar that crosses the average for the first time fails the first clause.

Short: C_{t−1} < SMMA_{t−1}, H_t ≥ SMMA_t − τ_m, C_t < SMMA_t.

## 6. Zone retest

$$\text{Long:}\ C_t > U_n \ \land\ Lo_t \le U_n + \tau_z$$

$$\text{Short:}\ C_t < L_n \ \land\ H_t \ge L_n - \tau_z$$

The wick has to reach within τ_z of the zone line. It can pierce the zone. It cannot close inside it.

## 7. Worked examples

**Long.** C = 4032.00, Lo = 4031.50, trend and regime pass.

- n = ⌊(4032.00 − 3978.86) / 17.50⌋ = ⌊3.0366⌋ = 3, so U₃ = 4031.36
- Close above: 4032.00 > 4031.36 ✓
- Retest: 4031.50 ≤ 4031.36 + 0.55 = 4031.91 ✓
- Distance: 4032.00 − 4031.36 = 0.64 < 49.5 ✓

**Short.** C = 4220.40, H = 4222.10, trend and regime pass.

- n = ⌈(4220.40 − 3976.22) / 17.50⌉ = ⌈13.953⌉ = 14, so L₁₄ = 4221.22
- Close below: 4220.40 < 4221.22 ✓
- Retest: 4222.10 ≥ 4221.22 − 0.55 = 4220.67 ✓
- Distance: 4221.22 − 4220.40 = 0.82 < 49.5 ✓

## 8. Timing and repainting

Every condition reads the bar's close, high or low, so the script can only decide once the bar has closed. On the bar still forming, a marker can appear and disappear as price moves. Alerts use `barstate.isconfirmed` and `alert.freq_once_per_bar_close`, so they fire once, at the close.

The higher-timeframe request adds one live-versus-history difference. With `lookahead_off`, a historical bar sees the 4H SMMA as of the last completed 4H bar. On the live bar, `request.security` returns the value from the 4H bar still forming. On a 2H chart, every other 2H bar closes halfway through a 4H bar, and on those bars the live 4H SMMA already includes the new 2H of price. A live alert and the historical marker for that bar can disagree when the close sits within a few points of the 4H SMMA.

Requesting the previous completed value with `[1]` and `lookahead_on` would remove the difference and change the signals. I have not re-validated that version, so the published script keeps the tested behaviour.

## 9. Alert stop and size

The alert message suggests a stop one offset s beyond the far line of the zone:

$$\text{SL}_{\text{long}} = L_n - s, \qquad \text{SL}_{\text{short}} = U_n + s$$

and a lot size for account A, risk fraction r and contract size c ounces per lot:

$$\text{lots} = \frac{\left\lfloor 100 \cdot \dfrac{A\,r}{|C - \text{SL}| \cdot c} \right\rfloor}{100}$$

With A = 10,000, r = 1%, c = 100 and the short example above, SL = 4223.86 + 11.2 = 4235.06, the stop distance is 14.66 and the size rounds down to 0.06 lots. The default s = 11.2 comes from the M15 EA. Set your own for other timeframes.
