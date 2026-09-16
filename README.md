# GCP Dataform Chicago Taxi Trips

This repository is a Google Cloud Dataform project for building a Chicago taxi trips analytics pipeline. It follows a standard medallion-style workflow:

- Source data are ingested and read from BigQuery and public datasets from Kaggle.
- Bronze tables store the raw data as-is.
- Silver tables apply cleaning, deduplication, and standardization.
- Gold tables model curated analytical data, dimensions, and KPI outputs.

## Repository structure

```
.
├── definitions/
│   ├── bronze/
│   │   ├── t_community_area.sqlx
│   │   ├── t_public_holiday.sqlx
│   │   └── t_taxi_trips.sqlx
│   ├── gold/
│   │   ├── dim_tables/
│   │   │   ├── t_dim_community_area.sqlx
│   │   │   ├── t_dim_company.sqlx
│   │   │   ├── t_dim_date.sqlx
│   │   │   └── t_dim_payment_type.sqlx
│   │   ├── fact_tables/
│   │   │   └── t_fact_chicago_taxi_trips.sqlx
│   │   └── kpi_charts/
│   │       ├── t_kpi_summary.sqlx
│   │       ├── t_peak_hours_area.sqlx
│   │       ├── t_public_holiday_trips.sqlx
│   │       ├── t_top_long_workers.sqlx
│   │       ├── t_top_payment_type.sqlx
│   │       └── t_top_tip_earners.sqlx
│   ├── silver/
│   │   ├── t_community_area.sqlx
│   │   ├── t_public_holiday.sqlx
│   │   └── t_taxi_trips.sqlx
│   └── source/
│       ├── chicago_community_area.sqlx
│       ├── taxi_trips.sqlx
│       └── us_public_holiday.sqlx
├── workflow_settings.yaml
├── README.md
└── .gitignore
```

## Source data origins

The project uses a combination of public cloud data and manual one-off datasets:

- Taxi trips: `bigquery-public-data.chicago_taxi_trips.taxi_trips`
- Chicago community area reference data: manually sourced from Kaggle (https://www.kaggle.com/datasets/sergejnuss/chicagocsv)
- US public holiday data: manually sourced from Kaggle (https://www.kaggle.com/datasets/jeremygerdes/us-federal-pay-and-leave-holidays-2004-to-2100-csv)

## Workflow schedule

The Dataform workflow is configured to run on a monthly schedule based on the `main` branch.

- Trigger timing: 16th of every month
- Time: 8:00 AM
- Timezone: GMT+8

This aligns with the monthly refresh cycle of the Chicago taxi trips dataset, which is updated on a monthly basis.

## Folder breakdown

### 1. definitions/source
This folder contains source table declarations. These are the upstream datasets referenced by later transformation steps.

- `taxi_trips.sqlx`: declares the Chicago taxi trip source table.
- `chicago_community_area.sqlx`: declares Chicago community area reference data.
- `us_public_holiday.sqlx`: declares USA public holiday reference data.

### 2. definitions/bronze
Bronze tables are a near-copy of the source data, stored in a raw layer for traceability and minimal transformation.

- `t_taxi_trips.sqlx`: reads raw taxi trip records into the bronze schema.
- `t_community_area.sqlx`: loads community area data into bronze.
- `t_public_holiday.sqlx`: loads public holiday data into bronze.

### 3. definitions/silver
Silver tables apply cleaning, deduplication, and standardization before analytics consumption.

- `t_taxi_trips.sqlx`: trims strings, normalizes fields, deduplicates by `unique_key`, and creates derived metrics such as trip minutes and trip meters.
- `t_community_area.sqlx`: standardizes community area metadata.
- `t_public_holiday.sqlx`: cleans and prepares holiday data for downstream joins.

### 4. definitions/gold/dim_tables
These are dimension tables used to enrich fact data and support dashboard/reporting use cases.

- `t_dim_date.sqlx`: date dimension for time-based analysis.
- `t_dim_company.sqlx`: taxi companies lookup table.
- `t_dim_payment_type.sqlx`: payment type lookup table.
- `t_dim_community_area.sqlx`: community-area lookup table.

### 5. definitions/gold/fact_tables
This folder contains the core fact tables used for analytical reporting.

- `t_fact_chicago_taxi_trips.sqlx`: the main fact table combining trip-level data with date, company, community area, and payment type dimensions.

### 6. definitions/gold/kpi_charts
This layer contains dashboard-ready KPI and chart datasets used for scorecards and visual analysis.

- `t_kpi_summary.sqlx`: high-level KPI summary dataset.
- `t_peak_hours_area.sqlx`: peak-hour analysis by community area.
- `t_public_holiday_trips.sqlx`: trips on public holidays.
- `t_top_long_workers.sqlx`: long worker summaries.
- `t_top_payment_type.sqlx`: most used payment types.
- `t_top_tip_earners.sqlx`: top tip-earner summaries.

 
## Question 1: Top 100 Tip Earners
 
**Definition used**: taxi IDs ranked by total tips earned, over the last 3 months of avalaible data. Since the data is only up to 2023, the calculation is anchored to `MAX(trip_start_timestamp)` in the dataset, not `CURRENT_DATE()`. 
 
**Assumptions**: cash payments are not logged in the system. This is because, 99% of trips that use cash as a payment method has 0 tips. So, this chart should be understood as top earners in recorded (non‑cash) tips, not necessarily their full tipping income.
 
---
 
## Question 2: Top 100 Overworkers
 
 **Definition used**: 
 1. trip_sequence: To build the sequence of trips for each taxi, the start and end time of every trip are selected. Then, using the LAG function, the end time of the previous trip is attached to the current one. This means each trip record now shows both its own start/end times and the prior trip’s end time. 
 2. trip_gap: Calculate how long is the gap between the trips.
 3. shift_flag: Since a break between shifts is supposed to be 8 hours, if the gap calculated between shifts is more than 8 hours, it is considered a new shift.
 4. shift_ids: Assign each trip a shift number based on these flags.
 5. shifts: Aggregate trips into shifts, calculating total duration and number of trips per shift.
 final: Summarize per taxi for total shifts, average shift hours, maximum shift hours, and count of long shifts. Select the Top 100 taxis with the highest number of long shifts.
 
**Assumptions**: A driver is considered a regular overworker if they log ≥ 5 long shifts (shifts longer than 12 hours). The 12‑hour threshold is based on the Municipal Code of Chicago (MCC Section 9‑112‑250), which sets maximum allowable driving hours for taxi operators. Breaks shorter than 8 hours are treated as part of the same continuous shift.
 
---
 
## Question 3: Public Holiday Impact on Trip Volume
 
 **Definition used**: daily trip counts grouped by date, year, and public holiday status, with each trip tagged as either “Public Holiday” (including holiday name) or “No Public Holiday” based on the date dimension.
 
---
