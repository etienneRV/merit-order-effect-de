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

Without a key, the notebook runs on synthetic data. It prints a warning banner, and every figure and table
title is tagged `[SYNTHETIC DATA]`.

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

## Results

<!-- fill in after reviewing actual notebook output -->
