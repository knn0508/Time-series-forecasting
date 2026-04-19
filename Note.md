# Forecasting Notes

## Notebook Status

| Notebook | Purpose | Status |
| --- | --- | --- |
| `Forecasting(AEP).ipynb` | Single-region hybrid forecasting for AEP | Executed end-to-end |
| `Forecasting.ipynb` | Multi-region panel forecasting with weather regressors | Implemented, but not fully benchmarked end-to-end yet because runtime is much higher |

## Modeling Flow

The forecasting workflow used in both notebooks follows the same overall structure:

1. Load hourly energy demand data and normalize it into the Prophet-friendly `ds` and `y` schema.
2. Inspect missing values, duplicates, outliers, and seasonality.
3. Handle missing hours and cap seasonal outliers.
4. Split the data chronologically into train, validation, and test sets.
5. Fit a Prophet baseline model to capture the main trend and recurring seasonal structure.
6. Compute Prophet residuals and learn those residual errors with XGBoost using lag, rolling, and calendar features.
7. Build the final hybrid forecast as:

   `Hybrid Forecast = Prophet Forecast + XGBoost Residual Correction`

This setup keeps Prophet focused on the broad time-series structure while XGBoost learns the remaining non-linear effects that Prophet misses.

## Why Prophet Was Chosen Over ARIMA/SARIMA

Prophet was chosen instead of ARIMA/SARIMA for several practical reasons:

- Prophet handles multiple seasonalities natively, especially daily, weekly, and yearly patterns, without needing extensive manual specification.
- It is more robust to missing data and irregularities, which matters for long hourly energy series.
- It exposes interpretable components such as trend, seasonality, and holiday effects, making the baseline easier to explain.
- ARIMA/SARIMA would likely require more manual tuning and becomes harder to maintain with long hourly data and multiple seasonal cycles.

For this problem, Prophet is a strong baseline because energy demand clearly contains overlapping seasonal patterns and calendar effects.

## Why XGBoost Was Chosen Over LSTM For Residuals

XGBoost was chosen instead of LSTM for the residual model because the residual-learning task is tabular and feature-driven:

- XGBoost trains much faster than LSTM, especially for repeated validation and tuning.
- It works naturally with lag features, rolling statistics, and calendar variables without sequence padding or window-building complexity.
- It is easier to interpret through feature importance plots.
- It usually performs very well on structured tabular forecasting residuals, especially when the baseline model already removed the main trend and seasonality.

In short, once Prophet extracts the coarse structure, the remaining job is better treated as a tabular correction problem than a deep sequence-modeling problem.

## AEP Notebook Findings

The AEP-only notebook is the one that has been fully run, so the observations below are based on actual notebook outputs.

### Prophet Baseline

- Best Prophet setting found on validation: `seasonality_mode=additive`, `changepoint_prior_scale=0.01`
- Test MAE: `1246.95`
- Test MAPE: `8.81%`

This shows that Prophet alone already captures the main recurring structure in AEP demand reasonably well.

### Hybrid Result

- Hybrid Test MAE: `550.86`
- Hybrid Test MAPE: `3.69%`
- MAPE improvement over Prophet baseline: `58.1%`

This is a large improvement and confirms that the Prophet residuals were not random noise. They still contained predictable structure that XGBoost was able to learn.

### What Feature Importance Revealed

The AEP dashboard shows that the strongest XGBoost residual features were:

1. `lag_24`
2. `hour`
3. `rolling_max_24`
4. `weekofyear`
5. `is_weekend`
6. `rolling_std_24`
7. `rolling_min_24`
8. `dayofweek`
9. `dayofyear`
10. `is_holiday`

This reveals several important things about energy consumption:

- The previous day's load at the same hour (`lag_24`) is the strongest driver by far, which means demand is highly persistent.
- Intraday schedule (`hour`) is extremely important, reflecting human activity cycles and daily operational rhythm.
- Recent short-window load behavior (`rolling_max_24`, `rolling_min_24`, `rolling_std_24`) matters because demand spikes and local volatility contain predictive information.
- Calendar structure (`weekofyear`, `dayofweek`, `is_weekend`, `is_holiday`, `dayofyear`) matters because energy demand changes with weekly routine, holidays, and seasonal positioning.

Overall, the feature importance suggests that energy demand is driven by a mix of persistence, time-of-day routine, weekly behavior, and seasonal context.

## Where The Hybrid Still Fails

For the AEP notebook, the main remaining failure cases are likely:

- unusual weather swings such as heat waves or cold snaps
- major holidays or holiday-adjacent behavior changes
- rare operational shocks or regime changes not well represented in training data

The key reason is that the AEP notebook does not include explicit weather regressors. The model has to infer weather effects indirectly from load lags and calendar structure, which works surprisingly well, but it is still indirect.

This is also consistent with the residual-learning result: if XGBoost can still improve the Prophet forecast this much, the residuals clearly contain structured information that Prophet alone did not model.

## Multi-Region Notebook Notes

The multi-region notebook (`Forecasting.ipynb`) extends the same hybrid logic to a concatenated panel with schema:

`ds | y | region`

Its design differs from the AEP notebook in several important ways:

- Prophet is fitted separately per region.
- XGBoost is trained as a global residual learner on the combined panel.
- Region identity is included in the XGBoost feature set through region-aware encoding.
- Weather regressors are now added through Open-Meteo.

### Added Weather Regressors

The multi-region notebook now includes explicit weather-related inputs, especially:

- `temperature_2m`
- `relative_humidity_2m`
- `heat_index`
- `HDH`
- `CDH`
- `temp_lag1h`
- `temp_lag3h`
- `temp_lag24h`
- `temp_rolling24h`

This is important because the AEP notebook already hinted that residuals were capturing weather effects indirectly. Adding temperature and humidity explicitly should reduce the need for XGBoost to recover that signal only through lagged demand.

### Expected Impact Of Explicit Weather Features

For the multi-region notebook, explicit weather features should help most on:

- summer peak demand days
- winter cold snaps
- sharp short-term demand shifts tied to uncomfortable temperature conditions

Since the AEP notebook already achieved a hybrid MAPE of `3.69%` without explicit weather, the main value of the added weather regressors is not just lowering average error further. It is also improving robustness on the hardest days and helping the all-regions notebook generalize better across different service territories.

Because the all-regions notebook has not been fully run yet, this remains an informed expectation rather than a confirmed benchmark. Still, it is reasonable to expect that explicit temperature and humidity regressors will help keep the hybrid model comfortably below a `10%` MAPE target at the broader multi-region level, especially when compared against a no-weather version of the same panel setup.

## Final Interpretation

The project supports a clear modeling conclusion:

- Prophet is a strong, interpretable baseline for hourly energy demand because it handles multiple seasonal patterns cleanly.
- XGBoost is an effective residual learner because the remaining forecast errors are structured and feature-driven.
- The AEP experiment proves the hybrid idea works in practice.
- The top residual features show that energy demand is highly persistent and strongly tied to daily and weekly human routine.
- Explicit weather variables are the most natural next step for reducing remaining errors, especially on abnormal days and when scaling from one region to many.
