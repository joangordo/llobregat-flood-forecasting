# Model card: Llobregat flood forecasting (v1.0)

## Model details

- **What it forecasts:** the 30-minute mean flow (m³/s) of the Llobregat River at Sant Joan
  Despí (ACA gauge LB-60, 4,908 km²) 1, 3, 6, 12 and 24 hours ahead, issued every 30 minutes.
- **Point forecast:** equal-weight mean of a LightGBM model of the relative flow change and a
  Ridge model of the absolute flow change (one pair of models per lead time).
- **Uncertainty:** LightGBM quantile models (5, 10, 90 and 95 %) calibrated with asymmetric
  conformalized quantile regression on a log scale, giving 80 % and 90 % intervals.
- **Warnings:** threshold = 99th percentile of the training flows: 87 m³/s for the evaluated
  model and 86 m³/s for the operational model (a high flow, not necessarily a flood).
  Pre-alert: upper bound of the 90 % interval above the threshold.
  Alert: point forecast above the threshold.
- **Author:** Joan Gordó Guardia (Universitat de Barcelona). **Version:** 1.0, November 2026.
- **License:** MIT (code). The training data have their own terms of use.

## Intended use

- Research, teaching and demonstration of data-driven flood forecasting.
- A supporting, experimental view of how the river may evolve in the next hours.

**Out of scope:** it is not an official warning system and must not be used to make
safety decisions. Official information comes from the Agència Catalana de l'Aigua and
Protecció Civil de Catalunya. The model has not been tested at other stations or rivers.

## Training data

- ACA open data, 5-minute records since 1996: flow at the target station and at six upstream
  gauges selected by coverage (Castellbell, Balsareny, Guardiola, Anoia at Sant Sadurní,
  Mediona and Cardener at Súria); daily outflow and volume of La Baells, La Llosa del Cavall
  and Sant Ponç reservoirs (previous complete day).
- ACA rain gauges, 5-minute records since 1996 (9 gauges in the basin, supplied by the ACA).
- Training period of the evaluated model: hydrological years 2001–2023 (October 2000 –
  September 2023), plus 2024 for the calibration of the intervals. The operational model is
  retrained with 2001–2025 and calibrated on 2026.
- Quality control removes spikes, dropouts, excursions, oscillations, stuck and saturated
  sensors, stretches stored in L/s, and days that do not match the official daily flow.
  The target is never imputed.

## Evaluation

Forward chaining by hydrological year: five outer folds (test years 2020–2024) and a final
2-year hold-out (2025–2026) that was evaluated once, after every decision had been taken.

| NSE | +1 h | +3 h | +6 h | +12 h | +24 h |
|---|---|---|---|---|---|
| Cross-validation 2020–2024 | 0.992 | 0.938 | 0.825 | 0.667 | 0.427 |
| Cross-validation, persistence | 0.983 | 0.899 | 0.765 | 0.549 | 0.193 |
| Hold-out 2025–2026 | 0.966 | 0.803 | 0.585 | 0.441 | 0.272 |
| Hold-out, persistence | 0.936 | 0.608 | 0.219 | −0.026 | −0.244 |
| Hold-out, Ridge alone (for reference) | 0.971 | 0.815 | 0.632 | 0.451 | 0.217 |

Peaks of the 32 hold-out episodes (flow above 87 m³/s):

| | +1 h | +3 h | +6 h | +12 h | +24 h |
|---|---|---|---|---|---|
| Median peak error | +0.3 % | −14.5 % | −27.6 % | −35.7 % | −46.6 % |
| Median peak delay | 0.5 h | 2.5 h | 6 h | 12 h | 23.5 h |

Warnings on the hold-out (32 episodes):

| | Issued before the observed crossing | Median lead time |
|---|---|---|
| Pre-alert (upper bound of the 90 % interval) | 29 / 32 | 3.5 h |
| Alert (point forecast) | 21 / 32 | 2 h |

| POD / FAR by lead time | +1 h | +3 h | +6 h |
|---|---|---|---|
| Alert | 0.84 / 0.07 | 0.50 / 0.38 | 0.38 / 0.43 |
| Pre-alert | 1.00 / 0.36 | 0.88 / 0.55 | 0.56 / 0.80 |

The thesis test period (June 2025 – May 2026) lies within the hold-out; on it, the NSE is
0.978 / 0.852 / 0.666 at +1 / +3 / +6 h (persistence: 0.945 / 0.659 / 0.333).

Intervals on the hold-out: coverage 0.86–0.90 (80 % interval) and 0.94–0.96 (90 %
interval); at high flows (above the 95th percentile) the coverage of the 80 % interval falls
from 0.82 at +1 h to 0.52 at +24 h.

## Limitations and risks

- **Late and low peaks at long lead times.** On the hold-out, from +6 h the forecast peak
  arrives about as late as persistence and 28 % or more too low. The model only sees rain that
  has already fallen. Floods driven by rain over the lower basin (1.5–4 h of travel time) cannot be
  anticipated beyond a few hours without a rainfall forecast.
- **Extrapolation.** The largest flood on record (Gloria, January 2020, about 1,470 m³/s at
  30-minute resolution) bounds what the trees have seen. The model predicts changes
  relative to the current flow, which extrapolates better than predicting the level, but a
  much larger event would still be outside the training range.
- **Calibration year.** The intervals of the evaluated model were calibrated on 2024, a
  drought year, and were somewhat too wide on the hold-out.
- **Data latency and gaps.** In real time, a missing upstream gauge degrades the forecast;
  the live app shows the data age and does not issue forecasts when the target station
  itself is missing.
- **Changing basin.** Reservoir operation, water abstraction or new gauges can change the
  relation between inputs and flow; the model should be re-evaluated on new years before
  being trusted again. The 2025–2026 hold-out has already been used and cannot serve for
  that.

## Reproducibility

`scripts/run_all.sh` rebuilds every result from the raw data; package versions are pinned in
`uv.lock` (`uv sync --frozen`). The operational model files record the library versions they
were trained with.
