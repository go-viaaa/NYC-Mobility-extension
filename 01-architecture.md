## 🏛 Architecture

The NYC Mobility Extension retains the existing **Medallion Architecture** while strengthening the pipeline with automated data-quality validation using **Great Expectations (GX)**.

The extended pipeline follows this flow:

```text
Source Data
    ↓
Preload Checks
    ↓
Bronze Layer
    ├── green_taxi_bronze
    ├── taxi_zone_bronze
    └── weather_bronze
    ↓
Bronze Quality Validation
    └── Great Expectations (GX)
    ↓
Silver Layer
    ├── green_taxi_silver
    ├── taxi_zone_silver
    └── weather_silver
    ↓
Silver Quality Validation
    └── Great Expectations (GX)
    ↓
Gold Layer
    ├── dim_date
    ├── dim_taxi_zone
    ├── dim_weather
    └── fact_trip
    ↓
Gold Quality Validation
    └── Great Expectations (GX)
