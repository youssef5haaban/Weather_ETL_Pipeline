# WeatherETL Pipeline

A multi-source ETL pipeline built with **Talend Open Studio 8.0.1** that extracts live weather data from the OpenWeatherMap API and enriches it with a static cities reference file, enforces data quality rules, and delivers clean and rejected outputs as structured CSV files.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Job_00_Master                               │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Subjob: Orchestration                                        │    │
│  │  Job_01_Extract ──OnSubjobOk──► Job_02_Transform_and_Load   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Subjob: 01_LoadContext + Monitoring (runs in parallel)       │    │
│  │  SRC_ContextProperties ──► LOAD_PipelineContext              │    │
│  │                ──OnComponentOk──► MONITOR_ErrorCatcher       │    │
│  │                                          ──► OUT_ErrorLog    │    │
│  └─────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                          Job_01_Extract                              │
│                                                                      │
│  01_LoadContext                                                       │
│   SRC_ContextProperties ──► LOAD_PipelineContext                     │
│                                    ──OnComponentOk──► LOG_JobStart   │
│                                         ├── OnComponentOk (order:1) ─► 02_ExtractWeatherAPI
│                                         └── OnComponentOk (order:2) ─► 03_ExtractCitiesCSV
│                                                                      │
│  02_ExtractWeatherAPI                                                 │
│   SRC_CityList ──► EXT_WeatherAPI ──► PARSE_WeatherJSON              │
│                                            ──► OUT_RawWeatherA       │
│                                               ──OnComponentOk──► LOG_CountRawA
│                                                                      │
│  03_ExtractCitiesCSV                                                  │
│   SRC_CitiesLookup ──► OUT_RawCitiesB                                │
│                              ──OnComponentOk──► LOG_CountRawB        │
│                                                                      │
│   LOG_CountRawA ──OnComponentOk──┐                                   │
│   LOG_CountRawB ──OnComponentOk──┴──► LOG_JobEnd                     │
│                                                                      │
│  04_Monitoring                                                        │
│   MONITOR_ErrorCatcher ──► OUT_ErrorLog                              │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│                   Job_02_Transform_and_Load                          │
│                                                                      │
│  01_LoadContext                                                       │
│   SRC_ContextProperties ──► LOAD_PipelineContext                     │
│                                    ──OnComponentOk──► LOG_JobStart   │
│                                         ├── OnSubjobOk (order:1) ──► 01_Normalize_Weather_Data
│                                         └── OnSubjobOk (order:2) ──► 02_Normalize_City_Data
│                                                                      │
│  01_Normalize_Weather_Data                                            │
│   SRC_RawWeathe ──raw_weather──► NORM_Weather                        │
│                    ──normalized_a_flow──► DQ_DeduplicateWeatherA     │
│                                               ──► OUT_StagingWeather │
│                                                                      │
│  02_Normalize_City_Data                                               │
│   SRC_RawCities ──raw_cities──► NORM_Cities                          │
│                    ──normalized_b_flow──► DQ_DeduplicateLookupB      │
│                                               ──► OUT_StagingCities  │
│                                                                      │
│  03_Enrich_and_Validate                                               │
│   SRC_StagingWeather ──weather_cleaned──► tMap_3                     │
│   SRC_StagingCities  ──cities_lookup (Lookup)──► tMap_3              │
│                                   ──OUT_EnrichedRows──► tFilterRow_1 │
│                                                │                     │
│                     is_rejected = false (row3) ├──► tReplicate_1     │
│                     is_rejected = true  (row1) └──► rejected_rows    │
│                                                                      │
│                          tReplicate_1 ──row5──► weather_clean        │
│                          tReplicate_1 ──row6──► tAggregateRow_1      │
│                                                    ──row7──► daily_summary
└──────────────────────────────────────────────────────────────────────┘

Final outputs:
  out/weather_clean.csv     — enriched, quality-checked rows
  out/rejected_rows.csv     — hard-rejected rows with rejection reason
  out/daily_summary.csv     — city-level aggregation
  logs/error_log.csv        — tLogCatcher runtime errors (all jobs, append mode)
```

### Pipeline stages

| Stage | Job | Subjob | What happens |
|---|---|---|---|
| Orchestration | Job_00_Master | — | Chains Job_01 → Job_02 via OnSubjobOk; loads master context; monitors errors |
| Context load | Job_01_Extract | 01_LoadContext | Reads Default.properties; triggers API and CSV extraction in parallel (order:1 / order:2) |
| API extract | Job_01_Extract | 02_ExtractWeatherAPI | Iterates city list via tFixedFlowInput; calls OpenWeatherMap per city; parses JSON; writes raw_weather_a.csv |
| CSV extract | Job_01_Extract | 03_ExtractCitiesCSV | Reads cities_lookup.csv; writes raw_cities_b.csv as-is |
| Monitoring | Job_01_Extract | 04_Monitoring | tLogCatcher captures all component errors; appends to error_log.csv |
| Context load | Job_02_Transform_and_Load | 01_LoadContext | Reloads context; triggers both normalization subjobs in parallel |
| Normalize weather | Job_02_Transform_and_Load | 01_Normalize_Weather_Data | Renames fields, casts types, derives city_code, deduplicates on city_name + forecast_dt; writes staging_a.csv |
| Normalize cities | Job_02_Transform_and_Load | 02_Normalize_City_Data | Trims fields, applies null defaults, deduplicates on city_code; writes staging_b.csv |
| Enrich & validate | Job_02_Transform_and_Load | 03_Enrich_and_Validate | LEFT JOIN (Lookup) on city_code; derives feels_like_delta, humidity_category, heat_index; computes data_quality_flag and is_rejected; routes to clean / reject / summary outputs |

---

## Sources

### Source A — OpenWeatherMap API (transactional)

Live weather observations fetched at runtime for 8 cities: Cairo, London, Paris, New York, Tokyo, Dubai, Lagos, Berlin.

**Endpoint:** `https://api.openweathermap.org/data/2.5/weather?q={city}&appid={api_key}&units=metric`

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
| city_code | String | Unique city identifier (join key) |
| city_name | String | City name |
| country | String | Country name |
| region | String | Geographic region |
| population_tier | String | Population category (e.g. Large, Mega) |
| utc_offset_hours | Integer | UTC offset for local time derivation |

### Join key

Both sources are joined on **city_code** using a **Lookup join** in tMap_3 (SRC_StagingCities is loaded as the lookup side — visible as the dashed orange connection in the job canvas). During normalization (NORM_Weather / tMap_1), a `city_code` value is derived from each weather row's `city_name` using the following rule:

- `"New York"` → `"NYC"` (special case)
- All other cities → first 3 characters of the city name, uppercased (e.g. `"Cairo"` → `"CAI"`)

The cities lookup source (NORM_Cities / tMap_2) trims and preserves the `city_code` column as-is from the reference file.

**Join type: LEFT JOIN** — all 8 weather rows are kept even if a city is missing from the lookup file. Unmatched rows will have null values for `region`, `country`, and other lookup fields, which the warning rules then flag.

---

## Normalization (Raw → Staging)

Both sources are normalized independently in their respective subjobs before the join.

### Weather source (Source A) — NORM_Weather (tMap_1) in subjob 01_Normalize_Weather_Data

| Raw field | Staged field | Transformation |
|---|---|---|
| dt | forecast_dt | Unix Integer → formatted date string `yyyy-MM-dd HH:mm:ss`; null → `""` |
| temp | temp_celsius | Cast to Float; null → `0.0` |
| feels_like | feels_like_celsius | Cast to Float; null → `0.0` |
| humidity | humidity_pct | Renamed, cast to Integer; null → `0` |
| pressure | pressure_hpa | Renamed, cast to Float; null → `0.0` |
| speed | wind_speed_ms | Renamed, cast to Float; null → `0.0` |
| description | weather_description | Trim whitespace; null → `"Unknown"` |
| all | cloud_cover_pct | Renamed, cast to Integer; null → `0` |
| city_name | city_code | Derived join key: `"New York"` → `"NYC"`; others → first 3 chars uppercased |

After tMap_1, **DQ_DeduplicateWeatherA** (tUniqRow) removes exact duplicate rows keyed on `city_name + forecast_dt` before writing to staging_a.csv.

### Cities source (Source B) — NORM_Cities (tMap_2) in subjob 02_Normalize_City_Data

All string fields trimmed. Null values receive safe defaults (`"Unknown Code"`, `"Unknown City"`, `"Unknown Country"`, `"Unknown Region"`, `"Unknown"`, UTC offset `0`). `city_name` is stored as `city_name_ref` in the staging schema to avoid column name collision after the join.

After tMap_2, **DQ_DeduplicateLookupB** (tUniqRow) removes duplicate `city_code` entries before writing to staging_b.csv.

---

## Derived Fields

Three new fields are computed in tMap_3 (subjob 03_Enrich_and_Validate) after the Lookup join:

| Field | Expression | Description |
|---|---|---|
| `feels_like_delta` | `(temp_celsius != null && feels_like_celsius != null) ? (temp_celsius - feels_like_celsius) : 0.0f` | Difference between actual and perceived temperature. Positive = feels colder than actual; negative = feels warmer. |
| `humidity_category` | `humidity_pct != null ? (humidity_pct > 70 ? "High" : "Normal") : "Unknown"` | Categorical humidity band. Values above 70% are classified as `"High"`, otherwise `"Normal"`. |
| `heat_index` | `temp_celsius == null ? "UNKNOWN" : (temp_celsius < 15 ? "COLD" : (temp_celsius <= 25 ? "MILD" : "HOT"))` | Three-tier temperature classification: below 15°C → `"COLD"`, 15–25°C → `"MILD"`, above 25°C → `"HOT"`. |

---

## Data Quality Rules

All DQ logic is evaluated inside tMap_3 in subjob **03_Enrich_and_Validate**. tFilterRow_1 then routes rows based on the `is_rejected` flag: passing rows (`is_rejected = false`) go to tReplicate_1 (→ weather_clean and daily_summary); rejected rows (`is_rejected = true`) go directly to rejected_rows.

### Error rules (hard reject)

Evaluated via the `is_rejected` boolean expression:

```
(temp_celsius < -80 || temp_celsius > 60) || (city_code == null)
```

| Rule | Condition | rejection_reason value |
|---|---|---|
| Temperature out of range | `temp_celsius < -80 OR temp_celsius > 60` | `"Temperature out of range: {value}"` |
| Missing city_code | `city_code IS NULL` | `"Missing city_code"` |

### Warning rules (soft flag)

Evaluated via the `data_quality_flag` string expression. Flagged rows are kept in the clean output.

```
(humidity_pct > 95 && cities_lookup.region == null)
    ? "High humidity – no region match"
    : (TalendDate.diffDate(forecast_dt, TalendDate.getCurrentDate(), "dd") > 6)
        ? "Forecast beyond 5-day window"
        : ""
```

| Rule | Condition | data_quality_flag value |
|---|---|---|
| High humidity unmatched | `humidity_pct > 95 AND region IS NULL` | `"High humidity – no region match"` |
| Stale forecast timestamp | `forecast_dt` more than 6 days from today | `"Forecast beyond 5-day window"` |

Rows that pass all warning rules have `data_quality_flag = ""` (empty string).

> **Note on rule priority:** The two warning conditions are evaluated as a chained ternary. If both conditions are true for the same row, only `"High humidity – no region match"` is assigned (it takes precedence). This is an implementation constraint of the current tMap expression and should be revisited if independent multi-flag support is needed.

---

## Outputs

| File | Contents | Idempotency |
|---|---|---|
| `out/weather_clean.csv` | Enriched, quality-checked rows including `feels_like_delta`, `humidity_category`, `heat_index`, and `data_quality_flag` | Overwrite on each run |
| `out/rejected_rows.csv` | Hard-rejected rows with `is_rejected = true` and `errorMessage` column | Overwrite on each run |
| `out/daily_summary.csv` | Aggregated summary grouped by `city_name` (`avg_temp_celsius`, `avg_humidity`, `record_count`) | Overwrite on each run |
| `logs/error_log.csv` | tLogCatcher output from all jobs: `moment`, `origin`, `type`, `message`, `code` | **Append** on each run |

### Idempotency approach

All data output files (`weather_clean`, `rejected_rows`, `daily_summary`) use `tFileOutputDelimited` with the **overwrite** action. Running the pipeline twice produces identical output files — no row duplication is possible because each run replaces the previous file entirely. This is the CSV equivalent of a truncate-and-reload strategy.

The error log (`error_log.csv`) uses **append** mode to preserve a full history of runtime errors across runs.

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
   WEATHERETL_PIPELINE/context/Default.properties
   ```
   Set the following values:
   ```
   api_key=YOUR_OPENWEATHERMAP_API_KEY
   api_base_url=https://api.openweathermap.org/data/2.5/weather
   cities=Cairo,London,Paris,New York,Tokyo,Dubai,Lagos,Berlin
   csv_path=in/cities_lookup.csv
   raw_a_path=temp/raw_weather_a.csv
   raw_b_path=temp/raw_cities_b.csv
   staging_a_path=temp/staging_a.csv
   staging_b_path=temp/staging_b.csv
   clean_output_path=out/weather_clean.csv
   reject_path=out/rejected_rows.csv
   summary_path=out/daily_summary.csv
   log_path=logs/error_log.csv
   ```

4. Create the required directories if they do not exist:
   ```
   mkdir temp out logs
   ```

5. Run the pipeline — open `Job_00_Master` in Talend Studio and press **Run**. Do not run the child jobs individually; the master job calls Job_01_Extract then Job_02_Transform_and_Load via tRunJob components chained with OnSubjobOk triggers.

---

## Assumptions and Trade-offs

**API pagination:** The OpenWeatherMap free tier returns one record per city per call. The pipeline iterates over the city list using a fixed-flow input (tFixedFlowInput). A production implementation would use the bulk forecast endpoint and handle pagination.

**Join key derivation:** Because no numeric city ID is shared between the API response and the lookup file, a `city_code` is derived from `city_name` during normalization. New York is handled as a special case (`"NYC"`); all other cities use their first three characters uppercased. This is fragile for cities sharing a three-letter prefix. In production, an authoritative city ID (e.g. a GeoNames ID) returned directly by the API would be a more reliable and collision-safe join key.

**Lookup join vs. standard join:** tMap_3 uses a Lookup join (SRC_StagingCities loaded into memory as the lookup side). This is efficient for the current scale since the cities reference file is small. At large scale, this would be replaced with a broadcast join in Spark to avoid loading the lookup table on a single node.

**Warning rule priority:** The `data_quality_flag` expression is a chained ternary, so only the first matching condition is recorded per row. A production implementation should support multiple concurrent flags per row, likely via a dedicated audit table or a delimited flag string.

**Static lookup file:** cities_lookup.csv is committed to the repository and loaded at runtime. In production this would be a database dimension table with a CDC mechanism to capture changes.

**No database target:** Outputs are written to CSV files using overwrite mode for idempotency. A production pipeline would write to a columnar store (e.g. Parquet on S3, or a data warehouse table) with partitioned overwrite by run date.

**Error handling:** tLogCatcher captures component-level errors in both jobs and appends them to error_log.csv. If the API is unavailable, the tHttpRequest component raises an exception which is caught and logged. The pipeline stops cleanly via the OnSubjobOk chain — Job_02 will not run if Job_01 fails.

**Deduplication:** tUniqRow is applied to each source after normalization to remove exact structural duplicates before the join. In production, a more robust deduplication strategy based on a primary key and ingestion timestamp would be used.

---

## Scale Question — What would change at 10 million rows per day?

**Storage:** CSV files would be replaced with Parquet, partitioned by `forecast_dt` date. This reduces scan costs on downstream queries and enables partition pruning.

**Processing engine:** Talend's single-node execution would be replaced with a distributed engine — Spark (via Talend Big Data) or dbt on a cloud warehouse. The tAggregateRow and tMap join would translate directly to Spark DataFrame operations or SQL transformations.

**Join strategy:** The current Lookup join (cities file loaded into tMap memory) would become a broadcast join in Spark — the small cities dimension is broadcast to all executors, avoiding a shuffle of the large weather dataset.

**API extraction:** 8 cities at one record each is trivial. At scale (thousands of cities or high-frequency polling), parallel HTTP calls with rate-limit handling and a dead-letter queue for failed calls would be required.

**Idempotency:** Overwrite-mode CSV is not viable at scale. A partitioned overwrite strategy (write to a dated partition, overwrite that partition on re-run) or an upsert keyed on `(city_code, forecast_dt)` would replace it.

**Orchestration:** The `tRunJob` parent job would be replaced with an Airflow DAG. Each child job becomes a task with explicit upstream dependencies, retry logic, SLA alerts, and a backfill mechanism for failed runs.

**Data quality:** At 10M rows, scanning every row in a single tMap becomes a bottleneck. Quality checks would be pushed into the warehouse as SQL assertions (e.g. dbt tests) running after load, with failed rows flagged in a dedicated audit table rather than a flat reject file. Multi-flag support per row would also be addressed at this stage.

---

## Project Structure

```
WeatherETL_Pipeline/
├── WEATHERETL_PIPELINE/                   # Talend project folder
│   ├── .settings/                         # Talend project settings
│   ├── context/                           # Context variables (Default.properties — not committed)
│   ├── metadata/                          # Talend metadata definitions
│   ├── process/                           # All jobs (Job_00_Master, Job_01_Extract, Job_02_Transform_and_Load)
│   └── talend.project                     # Talend project descriptor
├── in/                                    # Input data folder
│   └── cities_lookup.csv                  # Source B — committed reference file
├── out/                                   # Final outputs (not committed)
│   ├── weather_clean.csv
│   ├── rejected_rows.csv
│   └── daily_summary.csv
├── temp/                                  # Intermediate staging files (not committed)
│   ├── raw_weather_a.csv
│   ├── raw_cities_b.csv
│   ├── staging_a.csv
│   └── staging_b.csv
├── params/                                # Pipeline parameters/config files
├── logs/                                  # Runtime logs (not committed)
│   └── error_log.csv
└── README.md
```

> **Note:** `WEATHERETL_PIPELINE/context/Default.properties` contains your API key. Add it to `.gitignore` and share credentials separately.
