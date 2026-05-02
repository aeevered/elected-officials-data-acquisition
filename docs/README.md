# Documentation

The purpose of this README is to outline a proposed approach to data acquisition and modeling for elected official data.

---

## Data Model

This is a proposed star schema dimensional data model for modeling elected official data.

An overview of how the tables are related can be found below.

### `dim_jurisdiction`

This table stores information about each county include name, location information, and standardized codes.

| Column | Description |
|--------|-------------|
| `id` | Primary key. |
| `state_fips` | Two-digit state FIPS. |
| `county_fips` | Three-digit county FIPS within state. |
| `county_fips_5` | Five-digit state+county FIPS (convenience). |
| `geoid` | Census GEOID for spatial joins (optional). |
| `county_name` | Canonical display name. |
| `state_postal_code` | Two-letter state. |
| `county_class` | county, parish, borough, independent_city, etc. |
| `row_effective_date` / `row_expiry_date` | Optional Slowly Changing Dimension Type 2 effective/expiration dates if boundaries or names change over time. |

### `dim_office_type`

This table stores the dimensional data for each office type. This is a useful dimension table given that different counties may have different offices.

| Column | Description |
|--------|-------------|
| `id` | Primary key. |
| `normalized_title` | e.g. Sheriff, County Clerk, Commissioner. |
| `office_type_description` | Optional longer description. |
| `source_id` | Foreign key to `dim_source`. |

### `dim_office_instance`

This table stores each instance of a specific office, for example in the case of multiple members of the same title.

| Column | Description |
|--------|-------------|
| `id` | Primary key. |
| `jurisdiction_id` | FK — county where this seat exists. |
| `office_type_id` | FK — normalized role for this seat. |
| `title_raw` | Title as printed on source sites. |
| `seat_number` | Multi-member at-large place number, if used. |
| `is_elected` | elected / appointed / unknown. |
| `term_length_years` | Term length in years if known from rules or sources. |
| `is_partisan` | partisan / nonpartisan / unknown. |
| `source_id` | Foreign key to `dim_source`. |
| `created_at` | Row created at datetime. |
| `updated_at` | Row updated at datetime. |

### `dim_person`

This dimensional table contains the actual person level details, including full name and suffix.

| Column | Description |
|--------|-------------|
| `id` | Primary key. |
| `full_name` | Display name. |
| `given_name` | Parsed given name (optional). |
| `middle_name` | Parsed middle name (optional). |
| `family_name` | Parsed family name (optional). |
| `suffix` | Jr., III, etc. |
| `source_id` | Foreign key to `dim_source`. |
| `created_at` | Row created at datetime. |
| `updated_at` | Row updated at datetime. |

### `dim_date`

Conformed calendar dimension: one row per calendar day. Used for tenure start/end on `fact_official_tenure`, snapshot date on `fact_jurisdiction_coverage_snapshot`, and other date-grain reporting.

| Column | Description |
|--------|-------------|
| `id` | Primary key (surrogate integer or `YYYYMMDD` key, per warehouse convention). |
| `calendar_date` | Calendar date (business natural key for the grain). |
| `year` | Four-digit calendar year. |
| `quarter` | Calendar quarter (1–4). |
| `month` | Month of year (1–12). |
| `day_of_month` | Day of month (1–31). |
| `day_of_week` | Day of week (encoding per convention, e.g. 1 = Monday … 7 = Sunday). |
| `is_weekend` | Optional flag for Saturday/Sunday. |

### `dim_source`

This table contains a row for each possible data source. This is useful as a tracking and monitoring feature, allowing the aability to track where the data has been source from.

This table can also serve as a reference of all sources data is being acquired from.

| Column | Description |
|--------|-------------|
| `id` | Primary key. |
| `source_name` | Human-readable label. |
| `source_type` | government_site, state_agency, third_party_aggregate, manual, other. |
| `trust_tier` | A_primary, B_curated, C_secondary. |
| `base_url` | Root URL if applicable. |

### `fact_official_tenure` (primary fact)

Primary fact table with each elected official with each row representing one elected official.

| Column | Description |
|--------|-------------|
| `id` | Surrogate primary key for the fact row. |
| `dim_office_instance_id` | Foreign key to seat-level row in `dim_office_instance`. |
| `dim_person_id` | Foreign key to person details in `dim_person`. |
| `jurisdiction_dim_key` | FK — county (`dim_jurisdiction`); repeated for filter performance; must stay consistent with seat. |
| `role_start_date_dim_key` | FK — `dim_date` for role start (unknown → unknown member or nullable FK per policy). |
| `role_end_date_dim_key` | FK — `dim_date` for role end; sentinel or NULL policy for incumbent. |
| `dim_source_record_id` | FK — main evidence (`dim_source_record`) for this tenure row. |
| `role_title_raw` | Degenerate — title string from source. |
| `tenure_status` | Degenerate — incumbent, former, acting, appointed_fill, unknown. |
| `party_affiliation` | Degenerate — party as of this tenure; optional `dim_party` later. |
| `entry_route` | Degenerate — general_election, special_election, appointment, succession, unknown. |
| `tenure_length_days` | Measure — days from start to end when both known; else null. |
| `is_current_flag` | Measure — 1 if incumbent per ETL rule, else 0. |
| `source_id` | Foreign key to `dim_source`. |
| `created_at` | Row created at datetime. |
| `updated_at` | Row updated at datetime. |

### `fact_jurisdiction_coverage_snapshot` (operational / QA)

**Grain:** one row per county per **snapshot date** (e.g., daily ETL).

| Column | Description |
|--------|-------------|
| `id` | Surrogate primary key for the fact row. |
| `jurisdiction_dim_id` | Foreign key to county (`dim_jurisdiction`). |
| `snapshot_date_dim_id` | Foreign key to as-of date for the metrics (`dim_date`). |
| `expected_office_count` | Measure — expected seats from catalog or heuristic. |
| `filled_office_count` | Measure — seats with a current tenure. |
| `coverage_pct` | Measure — `filled / expected` when expected > 0. |
| `last_successful_fetch_ts` | Degenerate — latest successful pull timestamp for that county. |
| `created_at` | Row created at datetime. |
| `updated_at` | Row updated at datetime. |

### Relationship summary (logical)

| Column | Description |
|--------|-------------|
| `dim_office_instance` | Parents: `dim_jurisdiction`, `dim_office_type`. Snowflake-style; star still connects via instance. |
| `dim_date` | Conformed calendar (one row per day); used for tenure dates and snapshot dates. |
| `fact_official_tenure` | Parents: `dim_office_instance`, `dim_person`, `dim_jurisdiction`, `dim_date` (start and end), `dim_source_record`. Primary star. |
| `dim_source_record` | Parent: `dim_source`. Provenance spine. |
| `fact_jurisdiction_coverage_snapshot` | Parents: `dim_jurisdiction`, `dim_date`. Coverage KPIs. |
| `fact_contact_snapshot` | Parents: `dim_office_instance`, `dim_date`, `dim_source_record`. Optional. |
| `fact_data_quality_event` | Optional parent: `dim_source_record`. Audit / QA. |

---

# Source Strategy

A combination of sources would be used to acquire the needed data for all local elected officials.

These sources could include:

1. Official State and Local Website data
2. Aggregated Data from Academic and Other Organizations (MIT Election Lab)
3.

Some questons to think about

MIT election data

# Collection approach

A combination of approaches

Some questons to think about

ETL pipelines would be built priimarily in python

Run in batch, since real time data will not be needed.

Inventory of data sources to pull from, use this to orchestrate the python jobs to each source.

Python extractors --> Object storage + warehouse ETL --> dbt / SQL transforms

This data might already be structured or it migth be semi or unstructured, including from PDF or html.

# Tradeoffs & Open Questions

1. How do we know the data is accurate? In cases of conflict between sources, what should happen?

2. How do we know that data is complete? In case of incompleteness, how should this be reflected in the data and when is the threshold for good enough?

3. How to best track changes in the data

In the data model section, I primarily defined the final (Gold) layer of the dimensional model, but likely there would be need to have intermediate (Bronze and Silver) data layers with initial data cleanup and standardization. However, the specifics of these layers would depend largely on what the source data looks like, so I have not included details for the initial proposal.

I think the monitoring will be one of the most challenging and important parts to this data acquisition process.
