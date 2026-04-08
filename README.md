# WeatherETL Pipeline

A multi-source ETL pipeline built with **Talend Open Studio 8.0.1** that extracts live weather data from the OpenWeatherMap API and enriches it with a static cities reference file, enforces data quality rules, and delivers clean and rejected outputs as structured CSV files.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Job_00_Master                           │
│  (single entry point — runs the full pipeline end to end)  │
└───────┬──────────────────────────┬──────────────────────────┘
        │ OnSubjobOk               │ OnSubjobOk
        ▼                          ▼
┌──────────────────┐    ┌──────────────────────────┐
│  Job_01_Extract  │    │  Job_02_Transform_and_Load│
└──────────────────┘    └──────────────────────────┘
        │                          │
        ▼                          ▼
  raw_weather_a.csv     staging_a.csv  ──► weather_clean.csv   (Target 1: clean rows)
  raw_cities_b.csv      staging_b.csv  ──► rejected_rows.csv   (Target 2: hard rejects)
                                       ──► daily_summary.csv   (Target 3: aggregation)
                                       ──► error_log.csv       (monitoring)

All errors → tLogCatcher → error_log.csv
```

### Pipeline stages

| Stage | Job | What happens |
|---|---|---|
| Extract | Job_01_Extract | Calls OpenWeatherMap API for 8 cities; loads cities_lookup.csv; writes raw files as-is |
| Raw → Staging | Job_02_Transform_and_Load (Subjob A) | Renames fields, casts types, deduplicates each source independently via DQ_DeduplicateWeatherA and DQ_DeduplicateLookupB |
| Staging → Enriched | Job_02_Transform_and_Load (Subjob B) | LEFT JOIN on city_name; derives temp_celsius and feels_like_celsius; logs record count |
| Quality enforcement | Job_02_Transform_and_Load (Subjob C) | Applies 2 error rules and 2 warning rules; routes rejects to rejected_rows sink |
| Aggregation | Job_02_Transform_and_Load (Subjob D) | tAggregateRow grouped by city_name producing avg_temp_celsius, avg_humidity, record_count |
| Load | Job_02_Transform_and_Load (Subjob E) | Writes all three outputs to final CSV destinations with overwrite for idempotency |
| Monitoring | All jobs | tLogCatcher captures errors from every component into error_log.csv |

---

## Sources

### Source A — OpenWeatherMap API (transactional)

Live weather observations fetched at runtime for 8 cities: Cairo, London, Paris, Tokyo, Dubai, Berlin, Istanbul, Nairobi.

**Endpoint:** `https://api.openweathermap.org/data/2.5/weather?q={city}&appid={api_key}`

**Raw fields returned:**

| Field | Type | Description |
|---|---|---|
| dt | Integer | Unix timestamp of observation |
| temp | Float | Temperature in Celsius |
| feels_like | Float | Feels-like temperature in Celsius |
| humidity | Integer | Humidity percentage |
| pressure | Integer | Atmospheric pressure (hPa) |
| speed | Float | Wind speed (m/s) |
| description | String | Weather condition description |
| all | Integer | Cloud coverage percentage |

### Source B — cities_lookup.csv (reference/dimensional)

Static reference file committed to the repository. Provides regional and demographic context for each city.

**Fields:**

| Field | Type | Description |
|---|---|---|
| city_code | String | Unique city identifier |
| city_name | String | City name (join key) |
| country | String | Country name |
| region | String | Geographic region |
| population_tier | String | Population category (e.g. Large, Mega) |
| utc_offset_hours | Integer | UTC offset for local time derivation |

### Join key

Both sources are joined on **city_name**. The weather API returns the city name in the response; the lookup file contains a matching `city_name` column. Both sides are uppercased during normalization to ensure a case-insensitive match.

**Join type: LEFT JOIN** — all 8 weather rows are kept even if a city is missing from the lookup file. Unmatched rows will have null values for `region`, `country`, and other lookup fields, which the warning rules then flag.

---

## Normalization (Raw → Staging)

Both sources are normalized independently before the join.

### Weather source (Source A)

| Raw field | Staged field | Transformation |
|---|---|---|
| dt | forecast_dt | Cast Integer → Date |
| temp | temp_celsius | API returns Celsius directly; cast to Float |
| feels_like | feels_like_celsius | API returns Celsius directly; cast to Float |
| humidity | humidity_pct | Renamed, cast to Integer |
| pressure | pressure_hpa | Renamed, cast to Integer |
| speed | wind_speed_ms | Renamed, cast to Float |
| description | weather_description | Trim whitespace; null → "unknown" |
| all | cloud_cover_pct | Renamed, cast to Integer |

### Cities source (Source B)

All string fields trimmed. `city_name` uppercased to match weather source. No nulls expected; rows with null `city_code` are rejected at the quality stage.

---

## Derived Fields

Two new fields are computed in the enrichment tMap after the join:

| Field | Expression | Description |
|---|---|---|
| temp_celsius | direct from API | Temperature already provided in Celsius; renamed for clarity |
| feels_like_celsius | direct from API | Feels-like temperature already in Celsius; renamed for clarity |

---

## Data Quality Rules

### Error rules (hard reject)

Rows violating these rules are written to `rejected_rows.csv` with a `rejection_reason` column. They do NOT appear in the clean output.

| Rule | Condition | rejection_reason value |
|---|---|---|
| Temperature out of range | `temp_celsius < -80 OR temp_celsius > 60` | `"Temperature out of range: {value}"` |
| Missing city_code | `city_code IS NULL` | `"Missing city_code"` |

### Warning rules (soft flag)

Rows violating these rules are kept in the clean output but carry a non-empty `data_quality_flag` column.

| Rule | Condition | data_quality_flag value |
|---|---|---|
| High humidity unmatched | `humidity_pct > 95 AND region IS NULL` | `"High humidity – no region match"` |
| Stale forecast timestamp | `observed_at` more than 6 days from today | `"Forecast beyond 5-day window"` |

Rows that pass all warning rules have `data_quality_flag = ""` (empty string).

---

## Outputs

| File | Contents | Idempotency |
|---|---|---|
| `output/weather_clean.csv` | Enriched, quality-checked rows with data_quality_flag column | Overwrite on each run |
| `output/rejected_rows.csv` | Hard-rejected rows with rejection_reason column | Overwrite on each run |
| `output/daily_summary.csv` | Aggregated summary grouped by city_name (avg_temp_celsius, avg_humidity, record_count) | Overwrite on each run |
| `logs/error_log.csv` | tLogCatcher output: moment, origin, type, message, code | Overwrite on each run |

### Idempotency approach

All output files use `tFileOutputDelimited` with the **overwrite** action. Running the pipeline twice produces identical output files — no row duplication is possible because each run replaces the previous file entirely. This is the CSV equivalent of a truncate-and-reload strategy.

---

## How to Run

### Prerequisites

- Talend Open Studio for Big Data 8.0.1
- Java 8 or 11
- An OpenWeatherMap free-tier API key (sign up at openweathermap.org)

### Setup

1. Clone the repository:
   ```
   git clone <repo-url>
   cd WeatherETL_Pipeline
   ```

2. Import the project into Talend Open Studio:
   - File → Import Items → select the project folder
   - Accept all dependencies

3. Configure context variables — open the context file at:
   ```
   contexts/Default.properties
   ```
   Set the following values:
   ```
   api_key=YOUR_OPENWEATHERMAP_API_KEY
   api_base_url=https://api.openweathermap.org/data/2.5/weather
   cities=Cairo,London,Paris,Tokyo,Dubai,Berlin,Istanbul,Nairobi
   csv_path=data/cities_lookup.csv
   raw_a_path=temp/raw_weather_a.csv
   raw_b_path=temp/raw_cities_b.csv
   staging_a_path=temp/staging_a.csv
   staging_b_path=temp/staging_b.csv
   clean_output_path=output/weather_clean.csv
   reject_path=output/rejected_rows.csv
   summary_path=output/daily_summary.csv
   log_path=logs/error_log.csv
   ```

4. Create the required directories if they do not exist:
   ```
   mkdir -p temp output logs
   ```

5. Run the pipeline — open `Job_00_Master` in Talend Studio and press **Run**. Do not run the child jobs individually; the master job calls Job_01_Extract then Job_02_Transform_and_Load via tRunJob components chained with OnSubjobOk triggers.

---

## Assumptions and Trade-offs

**API pagination:** The OpenWeatherMap free tier returns one record per city per call. The pipeline iterates over the city list using a fixed-flow input. A production implementation would use the bulk forecast endpoint and handle pagination.

**Join key normalization:** city_name is used as the join key because no numeric city ID is shared between the API response and the lookup file. Both sides are uppercased before joining to prevent case mismatch failures. In production, an authoritative city ID (e.g. a GeoNames ID) would be a more reliable key.

**Static lookup file:** cities_lookup.csv is committed to the repository and loaded at runtime. In production this would be a database dimension table with a CDC mechanism to capture changes.

**No database target:** Outputs are written to CSV files using overwrite mode for idempotency. A production pipeline would write to a columnar store (e.g. Parquet on S3, or a data warehouse table) with partitioned overwrite by run date.

**Error handling:** tLogCatcher captures component-level errors. If the API is unavailable, the tHttpRequest component raises an exception which is caught and logged. The pipeline stops cleanly via the OnSubjobOk chain — Job_02 will not run if Job_01 fails.

**Deduplication:** tUniqueRow is applied to each source after normalization to remove exact structural duplicates before the join. In production, a more robust deduplication strategy based on a primary key and ingestion timestamp would be used.

---

## Scale Question — What would change at 10 million rows per day?

**Storage:** CSV files would be replaced with Parquet, partitioned by `observed_at` date. This reduces scan costs on downstream queries and enables partition pruning.

**Processing engine:** Talend's single-node execution would be replaced with a distributed engine — Spark (via Talend Big Data) or dbt on a cloud warehouse. The tAggregateRow and tMap join would translate directly to Spark DataFrame operations or SQL transformations.

**Join strategy:** A broadcast join (cities lookup is small, weather is large) would be used in Spark to avoid a shuffle. The lookup file would be broadcast to all executors rather than shuffled.

**API extraction:** 8 cities at one record each is trivial. At scale (thousands of cities or high-frequency polling), parallel HTTP calls with rate-limit handling and a dead-letter queue for failed calls would be required.

**Idempotency:** Overwrite-mode CSV is not viable at scale. A partitioned overwrite strategy (write to a dated partition, overwrite that partition on re-run) or an upsert keyed on `(city_name, observed_at)` would replace it.

**Orchestration:** The `tRunJob` parent job would be replaced with an Airflow DAG. Each child job becomes a task with explicit upstream dependencies, retry logic, SLA alerts, and a backfill mechanism for failed runs.

**Data quality:** At 10M rows, scanning every row in a single tMap becomes a bottleneck. Quality checks would be pushed into the warehouse as SQL assertions (e.g. dbt tests) running after load, with failed rows flagged in a dedicated audit table rather than a flat reject file.

---

## Project Structure

```
WeatherETL_Pipeline/
├── data/
│   └── cities_lookup.csv                  # Source B — committed reference file
├── contexts/
│   └── Default.properties                 # All config and secrets (not committed — see .gitignore)
├── process/
│   ├── Job_01_Extract/                    # Extract folder
│   └── Job_02_Transform_and_Load/         # Transform + Load folder
├── Job_00_Master/                         # Master orchestration job
├── temp/                                  # Intermediate staging files (not committed)
│   ├── raw_weather_a.csv
│   ├── raw_cities_b.csv
│   ├── staging_a.csv
│   └── staging_b.csv
├── output/                                # Final outputs (not committed)
│   ├── weather_clean.csv
│   ├── rejected_rows.csv
│   └── daily_summary.csv
├── logs/
│   └── error_log.csv
└── README.md
```

> **Note:** `contexts/Default.properties` contains your API key. Add it to `.gitignore` and share credentials separately.
