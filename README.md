# Merit-order effect in the German day-ahead electricity market (DE-LU, 2019–2025)

## Research question

Does a higher renewable generation share reduce German day-ahead electricity prices via merit-order
displacement, and has this effect strengthened over 2019–2025? The renewable-share coefficient is estimated
on the full sample and compared across two sub-periods, 2019–2021 and 2022–2025.

## Data source

[ENTSO-E Transparency Platform](https://transparency.entsoe.eu/), bidding zone **DE-LU**, 2019-01-01 to
2025-12-31 (Europe/Berlin), accessed with [`entsoe-py`](https://github.com/EnergieID/entsoe-py):

| Series | `entsoe-py` call | Used as |
|---|---|---|
| Day-ahead price (EUR/MWh) | `query_day_ahead_prices` | `price` |
| Actual generation per production type (MW) | `query_generation` | Wind Onshore + Wind Offshore, Solar, total generation (sum of all types, "Actual Aggregated") |
| Actual total load (MW) | `query_load` | `load` |

- Raw pulls are cached per series and calendar year in `data/<series>_<year>.parquet` (gitignored);
  re-runs read the cache instead of calling the API.
- Timestamps are converted to UTC and resampled to hourly means (this aggregates the 15-minute generation and
  load series and the 15-minute day-ahead prices from October 2025). Calendar features use Europe/Berlin local
  time, so CET/CEST transition days contain 23 or 25 hourly observations.
- Negative prices are kept unchanged in all estimations.
- Hours with a missing model input are dropped, and the notebook reports how many.

### Running it

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env          # then put your ENTSO-E API key in .env
jupyter notebook notebook.ipynb   # Kernel -> Restart & Run All
```

An ENTSO-E API key is required. Without a valid `ENTSOE_API_KEY` in `.env`, the notebook fails immediately
with a clear error rather than falling back to anything.

## Methodology

**Features**

- `renewable_share = (wind_onshore + wind_offshore + solar) / total_generation` (fraction between 0 and 1)
- `residual_load = actual_load − (wind + solar)`, in GW
- Month-of-year and hour-of-day dummies (local time, entered as `C(month)` and `C(hour)`; the reference
  categories are January and hour 0)
- `negative_price` = 1 if the day-ahead price is below 0 (descriptive flag, not a regressor)

**Model.** OLS estimated with `statsmodels`:

```
price_t = β0 + β1·renewable_share_t + β2·residual_load_t + Σ month dummies + Σ hour dummies + ε_t
```

Standard errors are Newey-West heteroskedasticity- and autocorrelation-consistent (HAC), with a Bartlett
kernel and a maximum lag of 168 hours (one week; set by `HAC_MAXLAGS`). For β1 the notebook reports the
coefficient, HAC standard error, t-statistic (z), p-value and R².

**Sub-period split.** The same specification is estimated separately on 2019–2021 and on 2022–2025
(Europe/Berlin calendar years, set by `SUBPERIODS`). The second window starts with the 2022 gas-price shock
and covers the subsequent period of higher renewable penetration. A comparison table reports β1, its HAC
standard error, t, p, N and R² for the full sample and for each sub-period.

**Robustness checks** (notebook Section 6). Each is estimated on the full sample and on both sub-periods;
only the `renewable_share` coefficient is reported.

1. Baseline specification, as above.
2. Total load (GW) instead of residual load. Residual load already subtracts wind and solar, so in the
   baseline β1 is the effect of renewable share holding residual load fixed.
3. Baseline with 336 HAC lags (two weeks).
4. Year-month fixed effects instead of month dummies, to absorb month-level shifts in fuel and CO₂ costs.
5. Optional: daily TTF gas (EUR/MWh) and EUA (EUR/t) prices as controls. This runs only if
   `data/fuel_prices.csv` (columns `date, ttf, eua`) is present; values are forward-filled over non-trading
   days, and the file is gitignored because it is not ENTSO-E data.

**Sub-period difference test.** Both periods are pooled, and `renewable_share` and the load control are
interacted with `post2022` (1 for 2022–2025). Month and hour effects are common to both periods, and
standard errors are HAC. The interaction coefficient is the difference in β1 between the periods. It is
reported for both the residual-load and the total-load specifications.

**Diagnostics.** The notebook reports the rows dropped for missing inputs (with the timestamps of any
missing prices), the number of 23- and 25-hour local days, the count and minimum of negative prices, and
any renewable share outside [0, 1].

## Results and discussion

Estimated on 61,362 hourly observations for the DE-LU bidding zone, 2019-01-01 to 2025-12-31.

| Specification (β₁ on `renewable_share`) | Full 2019–2025 | 2019–2021 | 2022–2025 |
|---|---|---|---|
| 1 Baseline (residual load control) | −64.0 (30.3)* | +46.6 (21.7)* | −220.3 (40.1)*** |
| 2 Total load instead of residual load | −177.3 (15.1)*** | −118.3 (14.1)*** | −312.1 (20.3)*** |
| 3 Baseline, HAC 336 lags | −64.0 (38.9) | +46.6 (23.7)* | −220.3 (52.1)*** |
| 4 Year-month fixed effects | −0.9 (7.9) | +5.4 (7.5) | +41.6 (12.1)*** |

HAC standard errors in parentheses. \* p < 0.05, \*\*\* p < 0.001. Pooled interaction test of
β₁(2022–2025) − β₁(2019–2021): −163.9 (35.1) with the residual-load control and −187.1 (24.6) with the
total-load control, both p < 0.001.

### What the data shows

The basic pattern is clear: hours with more wind and solar have lower prices. This holds across the whole
range of renewable shares in both sub-periods, and it is much stronger in the later years. In 2022–2025,
average prices fall from about 240 EUR/MWh in the hours with the least renewable generation to roughly zero
in the hours where renewables supply more than 80 percent of output. In 2019–2021 the same curve runs from
about 100 EUR/MWh down to just below zero. The interaction test confirms that the two periods really do
differ, whichever load control is used.

How large the effect is, however, depends heavily on how the model is set up.

- **Spec 1** controls for residual load, which is load minus wind and solar. Because that control already
  subtracts renewable output, the renewable-share coefficient picks up only what is left over, and it even
  turns positive in 2019–2021.
- **Spec 2** uses total load instead. This leaves the displacement effect in the coefficient, and the result
  is negative in every period.
- **Spec 4** adds a separate dummy for each calendar month of the sample, which soaks up the swings in gas
  and carbon prices. Almost the entire full-sample effect disappears, and the share of price variation the
  model explains rises from 20 percent to 83 percent.

The last point is the important one. Most of the gap between the two sub-periods in specs 1 to 3 comes from
fuel being far more expensive after 2021, not from a lasting change in how the market's merit order works.

### Interpretation

Prices in Germany are usually set by the most expensive plant running in a given hour, typically gas or
coal. Every extra MWh of wind or solar, which costs almost nothing to produce, pushes that plant out of the
market. The saving therefore depends on what the displaced plant costs to run. That is why the effect looks
so large in 2022–2025, when gas and carbon prices peaked, and why it largely disappears once those cost
swings are accounted for.

For the market, this means renewables push down the very prices they earn, and they do so most strongly when
prices are high in the first place. That matters for how much wind and solar farms actually capture per MWh
sold, for the pricing of contracts for difference and power purchase agreements, and for the business case
for batteries and flexible demand. In 2,050 hours, or 3.3 percent of the sample, prices went negative, and
at the extreme they hit the market floor of −500 EUR/MWh. At that point the issue is no longer just lower
revenue, but the risk of being curtailed outright.

### Limitations

These results show association, not causation.

- Wind and solar output depends on the weather, so it is largely unaffected by the price. But the
  denominator of `renewable_share` is total generation, which does respond to prices through cross-border
  trade, so the variable is not fully exogenous.
- The model uses only same-hour values. It leaves out price dynamics, storage, interconnector flows and
  plant availability. A Durbin-Watson statistic of 0.031 shows the residuals remain strongly autocorrelated;
  the HAC standard errors account for this, but the model itself does not.
- The baseline has no direct gas or carbon price control. The notebook includes an optional specification
  for one once that data is supplied.
- The 2022–2025 window mixes the gas price shock with continued growth in renewable capacity, and the model
  cannot separate the two.

A cleaner estimate of the structural merit-order effect would control for fuel and carbon prices directly,
instrument the renewable share with weather data, and model the price dynamics explicitly.
