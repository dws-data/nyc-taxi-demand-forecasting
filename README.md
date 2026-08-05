# NYC Taxi Demand Forecasting

An end-to-end data analytics project on NYC TLC trip data: ingestion, exploratory analysis, a Power BI dashboard, and (upcoming) time-series demand forecasting — built entirely on Databricks Free Edition.

## Overview

NYC publishes monthly trip-record data for Yellow taxis, Green (boro) taxis, and High-Volume For-Hire Vehicles (HVFHV — Uber/Lyft/Via). This project pulls that data (2019–2025), aggregates it to hourly demand per vehicle type, explores how the market has shifted over time (including the COVID-19 crash and recovery), and visualizes it in a Power BI dashboard. The next phase adds forecasting models (ARIMA, Prophet, XGBoost) to predict future demand.

## Tech stack

- **Databricks Free Edition** — PySpark, Delta Lake (managed tables), MLflow
- **Power BI Desktop** — connected directly to a Databricks serverless SQL warehouse (no CSV exports)
- **Python** — pandas, matplotlib for exploratory analysis

## Data

- **Source:** [NYC TLC Trip Record Data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) — public Parquet files, one per vehicle type per month
- **Range:** 2019–2025, Yellow + Green + HVFHV (~250 source files, ~40–50 GB raw)
- **Storage:** aggregated to a single `hourly_demand` Delta table (`hour`, `vehicle_type`, `trip_count`) — raw files are never persisted, only streamed through and aggregated, to stay within Databricks Free Edition's 10 GB storage quota. See `PIPELINE.md` for the full data flow.

## Project status

| Phase | Status |
|---|---|
| 1 — Ingest (`01_ingest`) | Complete |
| 2 — Exploratory analysis (`02_explore`) | Complete |
| 2b — Power BI dashboard (Pages 1–3) | Complete |
| 3 — Feature engineering | Ongoing |
| 4 — Modelling (ARIMA, Prophet, XGBoost) | Not started |

## Dashboard

Three pages built so far:

| Page | Content |
|---|---|
| Title | Project overview, data source, date range |
| Market Overview | Trips over time by vehicle type, market share %, COVID-19 crash/recovery zoom-in |
| Demand Patterns | Average trips by hour of day, day of week, month of year |

![Market Overview](screenshots/power_bi/final/market_overview.png)
![Demand Patterns](screenshots/power_bi/final/demand_patterns.png)

## Repo structure

```
notebooks/          Databricks notebooks (01_ingest, 02_explore, ...)
powerbi/            .pbix dashboard file + theme.json
screenshots/         Final exploratory and dashboard chart images
PIPELINE.md          Full data flow / architecture reference
```

## Roadmap

- `03_features` — calendar features, lag features, rolling stats, train/test split (train 2019–2024, test 2025)
- `04_modelling` — ARIMA, Prophet, and XGBoost across 4 forecast targets (Yellow, Green, HVFHV, Combined), tracked with MLflow
- Forecast results added to the Power BI dashboard as new pages
