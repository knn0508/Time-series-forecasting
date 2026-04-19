# Time-Series Forecasting

This project builds hybrid electricity load forecasting models with **Prophet + XGBoost**.

The repository contains:

- an **AEP-only notebook** that has already been run end-to-end
- a **multi-region notebook** that extends the same workflow to all regional datasets in `archive/`
- supporting visualizations and notes documenting the modeling choices

## Project Structure

- `Forecasting(AEP).ipynb`
  Single-region workflow for AEP. This is the notebook with completed benchmark results.
- `Forecasting.ipynb`
  Multi-region panel workflow using the schema `ds | y | region`, plus Open-Meteo weather regressors.
- `Note.md`
  Modeling rationale, AEP results, feature-importance interpretation, and limitations.
- `archive/`
  Hourly regional load CSV files used by the notebooks.
- `Visualization/`
  Saved charts and dashboard outputs generated from the notebooks.

## Modeling Approach

Both notebooks follow the same high-level pattern:

1. Load hourly energy demand data.
2. Clean duplicates, handle missing hours, and cap seasonal outliers.
3. Split data chronologically into train, validation, and test sets.
4. Fit a **Prophet** baseline model.
5. Compute Prophet residuals.
6. Train **XGBoost** on lag, rolling, and calendar features to predict those residuals.
7. Combine both outputs into the final hybrid forecast:

   `Hybrid Forecast = Prophet Forecast + XGBoost Residual Correction`

In the multi-region notebook, weather regressors from **Open-Meteo** are also added.

## Why This Hybrid Model

- **Prophet** is used as the baseline because it handles multiple seasonalities cleanly and is easy to interpret.
- **XGBoost** is used for residual learning because it is fast, works very well with tabular lag features, and avoids the extra complexity of LSTM sequence modeling.

More detail is documented in [Note.md](./Note.md).

## Current Results

### AEP Notebook

The completed AEP run produced:

- Prophet Test MAE: `1246.95`
- Prophet Test MAPE: `8.81%`
- Hybrid Test MAE: `550.86`
- Hybrid Test MAPE: `3.69%`
- MAPE improvement: `58.1%`

### Multi-Region Notebook

The multi-region notebook has been implemented, including:

- concatenated panel format: `ds | y | region`
- per-region Prophet modeling
- global XGBoost residual learning
- Open-Meteo weather regressors:
  - `temperature_2m`
  - `relative_humidity_2m`
  - `heat_index`
  - `HDH`
  - `CDH`
  - temperature lag and rolling features

This notebook has **not been fully benchmarked end-to-end yet** because it takes longer to run.

## Requirements

The notebooks use the following Python packages:

- `pandas`
- `numpy`
- `matplotlib`
- `seaborn`
- `plotly`
- `prophet`
- `xgboost`
- `scikit-learn`
- `optuna`
- `mlflow`
- `holidays`
- `statsmodels`
- `openmeteo-requests`

The multi-region notebook includes a notebook cell to verify the weather package:

```python
!pip show openmeteo-requests
```

## How To Run

1. Open the repository in Jupyter.
2. Run `Forecasting(AEP).ipynb` if you want the faster, already-validated single-region workflow.
3. Run `Forecasting.ipynb` if you want the multi-region version with weather regressors.
4. Review [Note.md](./Note.md) for the written analysis and interpretation.

## Notes

- The AEP notebook is the best reference if you want confirmed output today.
- The multi-region notebook is the more complete production-style workflow, but it is more computationally expensive.
- The remaining errors are most likely concentrated around rare events, strong weather anomalies, or special holiday behavior.
