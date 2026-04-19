# Spatiotemporal Lightning Analytics

> Production-grade geospatial lightning analytics on PostgreSQL 17 + PostGIS.  
> 802ms → 167ms · 1M rows · Time-based partitioning · Spatial joins · Python ETL

---

<!-- add a picture here — Superset dashboard showing scattered lightning points across India -->

---

## What & Why

Lightning strike data arrives in high volumes, unevenly distributed across time and geography. A naive flat-table schema collapses under aggregation queries at scale — full sequential scans, no spatial awareness, no pre-computed summaries.

This platform solves that. It stores 1M+ strike events in a partitioned PostGIS schema, spatially joins them to state boundaries, and pre-aggregates them into materialized views so common queries return in milliseconds instead of seconds. The ETL pipeline handles real-world messiness: out-of-bounds coordinates, bad timestamps, duplicate IDs, and partial data.

Designed for weather analytics, research, and government monitoring use cases where query speed and data quality both matter.

---

## Highlights

- 🚀 Reduced time-series query time from **802ms → 167ms** via partition pruning (4.8× faster)
- 🗺️ PostGIS spatial join matching **344k+** strike events to state boundaries
- 🧱 Range partitioning across 6 years with automatic partition routing
- ⚙️ Materialized views pre-aggregating 1M rows into **57,916** daily-state summaries
- 🐍 Python ETL pipeline with validation, deduplication, and 10k-row batch inserts
- 🔁 Alembic schema migrations with full upgrade/downgrade support
- 📊 Apache Superset dashboard with state + time-range filters on a live map

---

## Quick Start

```bash
git clone https://github.com/your-username/lightning-data-platform
cd lightning-data-platform

# 1. Create the database
psql -U postgres -c "CREATE DATABASE lightning_db;"

# 2. Apply all migrations (schema + indexes + materialized views)
python -m alembic upgrade head

# 3. Load state boundary shapefile
shp2pgsql -s 4326 shapefiles/India_State_Boundary.shp india_states_temp | psql -U postgres -d lightning_db

# 4. Generate 1M synthetic rows (~95 seconds)
python ingestion/synthetic_generator.py

# 5. Run spatial join + refresh materialized views
psql -U postgres -d lightning_db -c "
  UPDATE lightning_strikes ls SET state_id = ist.state_id
  FROM india_states ist WHERE ST_Within(ls.geom, ist.geom);
  SELECT refresh_lightning_views();
"

# 6. Ingest real data
python ingestion/ingest.py --file your_file.csv
```

**Requirements:** PostgreSQL 17, PostGIS, Python 3.10, psycopg2, Alembic, NumPy

---

## Performance Benchmarks

Benchmarked with `EXPLAIN ANALYZE` on 1,000,000 rows.  
**Environment:** Windows 10, PostgreSQL 17, local disk, warm cache.  
**Method:** `SET enable_indexscan=off; SET enable_bitmapscan=off;` to force sequential scans for the BEFORE baseline, then reset to defaults for AFTER.

| Query | Before | After | Improvement |
|---|---|---|---|
| Time range aggregation (single year) | 802ms | 167ms | **4.8× faster** |
| Composite filter (time + state) | 53ms | 69ms | Index scan confirmed |
| High intensity filter (multi-year) | 255ms | 161ms | **1.6× faster** |
| Grouped aggregation on MV | 308ms | 184ms | **1.7× faster** |

> Planning time on multi-year query: 16.7ms → 0.67ms **(25× faster)**

Raw `EXPLAIN ANALYZE` output: [`benchmarks/before_optimization.md`](benchmarks/before_optimization.md) · [`benchmarks/after_optimization.md`](benchmarks/after_optimization.md)

---

## Architecture

```
CSV / Synthetic Data
        │
        ▼
  ingest.py / synthetic_generator.py
  (validate → dedup → batch insert)
        │
        ▼
PostgreSQL 17 + PostGIS
├── lightning_strikes  [PARTITIONED BY RANGE(timestamp)]
│   ├── 2021 · 2022 · 2023 · 2024 · 2025 · 2026 · default
├── india_states  [36 state polygons, GiST indexed]
├── lightning_master_mv          (1,000,000 rows)
└── lightning_daily_summary_mv     (57,916 rows)
        │
        ▼
  Apache Superset 6.0
  (scatter map · state filter · time-range filter)
```

---

## Key Features

### Time-Based Partitioning
`lightning_strikes` is partitioned by `RANGE(timestamp)` across yearly child tables. A query scoped to 2023 scans only `lightning_strikes_2023` (166k rows) instead of all 1M — the largest single performance gain in the project.

### Indexing Strategy

| Index | Type | Purpose |
|---|---|---|
| `idx_lightning_timestamp_state` | Composite B-tree | Time + state queries |
| `idx_lightning_high_intensity` | Partial B-tree | High-radiance filter only (smaller index) |
| `idx_lightning_geom` | GiST | Spatial containment / distance |
| `idx_lightning_flash_type` | B-tree | Flash type filtering (CG / IC / CC) |
| `idx_states_geom` | GiST | State boundary spatial joins |

### Materialized Views
Two MVs pre-compute expensive aggregations and are refreshed concurrently (zero downtime) via a stored function:

```sql
SELECT refresh_lightning_views();
-- lightning_master_mv:        1,000,000 rows
-- lightning_daily_summary_mv:    57,916 rows
```

### Spatial Join (PostGIS)
Two-pass join: exact `ST_Within` containment first, then a 0.05° `ST_DWithin` boundary sweep for edge cases.

```
Pass 1 (ST_Within):   333,357 rows matched
Pass 2 (ST_DWithin):   11,022 rows matched
Total:                344,379 / 1,000,000 (34.4%)
```

Remaining 65.6% fall outside land boundaries (ocean, neighboring countries) — expected for data generated across India's full bounding box rectangle.

### ETL Pipeline
`ingest.py` runs full validation before any insert:
- Coordinates within India bounds (lat 6.5–35.5, lon 68.0–97.5)
- `flash_type` ∈ {CG, IC, CC}, `quality_flag` ∈ {0, 1, 2}, `radiance` ≥ 0
- Valid ISO timestamp, valid integer `event_id`
- Deduplication check against existing IDs
- Rejected rows written to `_rejected.csv` with error reasons

```bash
python ingestion/ingest.py --file your_file.csv
# Inserted: 500 | Dupes: 0 | Rejected: 0 | Time: 2.1s
```

### Schema Migrations (Alembic)
```
<base> → create_lightning_strikes_partitioned
       → add_indexes
       → add_materialized_views  (head)
```

```bash
python -m alembic upgrade head   # apply
python -m alembic downgrade -1   # rollback
python -m alembic history        # audit trail
```

---

## Sample Output

<!-- add a picture here — SQL Lab query result showing event_id, latitude, longitude, state_name, radiance columns -->

**Top states by strike count (after spatial join):**

| State | Strikes |
|---|---|
| Rajasthan | 36,379 |
| Madhya Pradesh | 32,319 |
| Maharashtra | 31,068 |
| Uttar Pradesh | 25,389 |
| Gujarat | 19,003 |

---

## Schema

```sql
lightning_strikes  (PARTITIONED BY RANGE timestamp)
├── event_id            bigint
├── latitude/longitude  numeric(9,6)
├── timestamp           timestamptz
├── radiance            numeric(10,4)
├── flash_type          text        -- CG / IC / CC
├── intensity_category  text        -- GENERATED STORED (Low / Medium / High)
├── quality_flag        smallint    -- 0 / 1 / 2
├── geom                geometry(Point, 4326)
└── state_id            FK → india_states
```

---

## Project Structure

```
lightning-data-platform/
├── database/
│   ├── schema.sql · partitions.sql · indexes.sql · materialized_views.sql
│   └── migrations/
│       ├── 5566ece10587_create_lightning_strikes_partitioned.py
│       ├── bdb65b394328_add_indexes.py
│       └── 48b26036eceb_add_materialized_views.py
├── ingestion/
│   ├── synthetic_generator.py   # 1M row generator with monsoon weighting
│   └── ingest.py                # ETL pipeline
└── benchmarks/
    ├── before_optimization.md
    └── after_optimization.md
```

---

## Tech Stack

**Database:** PostgreSQL 17 · PostGIS  
**Language:** Python 3.10  
**Libraries:** psycopg2 · NumPy · Alembic · SQLAlchemy  
**Visualization:** Apache Superset 6.0
