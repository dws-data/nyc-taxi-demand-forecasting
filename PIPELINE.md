# Pipeline Architecture — NYC Taxi Demand Forecasting

## Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│  SOURCE: nyc.gov                                            │
│  NYC TLC Trip Record Data — Parquet, one file per month     │
│  Yellow (84 files) · Green (84 files) · HVFHV (83 files)   │
│  2019–2025 · ~40–50 GB total                                │
└─────────────────────┬───────────────────────────────────────┘
                      │ requests (HTTP download)
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  /tmp  (local disk on Databricks driver VM)                 │
│  One file at a time — not counted against 10 GB quota       │
│  Deleted immediately after processing                        │
└─────────────────────┬───────────────────────────────────────┘
                      │ spark.read.parquet()
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  NOTEBOOK 1: 01_ingest                                      │
│                                                             │
│  For each file:                                             │
│  1. Check ingest_log — skip if already done                 │
│  2. Download to /tmp                                        │
│  3. Spark reads file, extracts pickup hour                  │
│     (tpep_ / lpep_ / pickup_datetime per type)              │
│  4. Count trips per hour → 3 columns:                       │
│     hour | vehicle_type | trip_count                        │
│  5. Append to hourly_demand Delta table                     │
│  6. Log success to ingest_log Delta table                   │
│  7. Delete /tmp file                                        │
└──────────┬──────────────────────────────┬───────────────────┘
           │                              │
           ▼                              ▼
  ┌─────────────────┐           ┌──────────────────┐
  │  ingest_log     │           │  hourly_demand   │
  │  Delta table    │           │  Delta table     │
  │                 │           │                  │
  │  vehicle_type   │           │  hour            │
  │  year           │           │  vehicle_type    │
  │  month          │           │  trip_count      │
  │  row_count      │           │                  │
  │  status         │           │  ~550k rows      │
  └─────────────────┘           │  (tiny — fits    │
  Checkpoint only               │  easily in 10GB) │
                                └────────┬─────────┘
                                         │
                    ┌────────────────────┤
                    │                    │
                    ▼                    ▼
┌───────────────────────────┐  ┌────────────────────────────┐
│  NOTEBOOK 2: 02_explore   │  │  NOTEBOOK 3: 03_features   │
│                           │  │                            │
│  Market shift story:      │  │  Adds to hourly_demand:    │
│  · Trips over time by     │  │  · Calendar features       │
│    type (stacked/overlay) │  │    (hour, dow, month,      │
│  · Market share % by type │  │     is_weekend, holiday)   │
│  · COVID crash + recovery │  │  · Lag features (1h–24h)   │
│  · Hourly/weekly patterns │  │  · Rolling means (24h, 1w) │
│  · No Delta table output  │  │  · Train/test split:       │
│    (visualisations only)  │  │    train=2019–24           │
└───────────────────────────┘  │    test=2025               │
                               └──────────┬─────────────────┘
                                          │
                                          ▼
                                 ┌─────────────────┐
                                 │  features       │
                                 │  Delta table    │
                                 └────────┬────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────┐
│  NOTEBOOK 4: 04_modelling                                   │
│                                                             │
│  Forecast targets: Yellow · Green · HVFHV · Combined (×4)  │
│  Train: 2019–2024   Test: 2025 (6 backtest windows)        │
│                                                             │
│  Backtest windows (2025):                                   │
│    Jan weekday · Jan weekend · Jul weekday · Jul weekend    │
│    Thanksgiving · New Year's Eve                            │
│                                                             │
│  Models (all logged to MLflow):                             │
│    ARIMA     — statsmodels — datetime + target only         │
│    Prophet   — Meta        — datetime + target only         │
│    XGBoost   — xgboost     — full feature matrix            │
│                                                             │
│  Metrics logged per run: MAE · RMSE · MAPE                  │
└──────────┬──────────────────────────────┬───────────────────┘
           │                              │
           ▼                              ▼
  ┌─────────────────┐           ┌──────────────────┐
  │  MLflow         │           │  forecast_output │
  │  Experiment UI  │           │  Delta table     │
  │                 │           │                  │
  │  Compare runs   │           │  hour            │
  │  across models  │           │  vehicle_type    │
  │  and targets    │           │  actual          │
  └─────────────────┘           │  predicted       │
                                │  model           │
                                └────────┬─────────┘
                                         │ Databricks connector
                                         │ (serverless SQL warehouse)
                                         ▼
                               ┌──────────────────────┐
                               │  POWER BI DESKTOP    │
                               │                      │
                               │  Connected to:       │
                               │  · hourly_demand     │
                               │    (EDA pages 1-3)   │
                               │  · forecast_output   │
                               │    (pages 4-6)       │
                               └──────────────────────┘
```

## Delta Tables Summary

| Table | Written by | Size | Purpose |
|---|---|---|---|
| `ingest_log` | Notebook 1 | Tiny | Checkpoint — tracks which files are done |
| `hourly_demand` | Notebook 1 | ~50 MB | Core time series — hour · type · trip_count |
| `features` | Notebook 3 | ~200 MB | Feature matrix for modelling |
| `forecast_output` | Notebook 4 | Tiny | Actuals + predictions for Power BI |

**Total storage:** well within 10 GB quota. Raw Parquet (~40–50 GB) is never persisted — processed and discarded file by file.

## Key Constraints

- **10 GB storage quota** (Databricks Free Edition) — reason raw tables are not persisted
- **DBFS root disabled** — all storage via `saveAsTable()` (managed Delta tables)
- **`/tmp`** — local driver VM disk, outside quota, holds one file at a time (~500 MB–1 GB max)
- **Serverless compute** — no classic clusters; cold start ~15–20s per session
- **Power BI connection** — direct Databricks connector via serverless SQL warehouse; no CSV export needed
