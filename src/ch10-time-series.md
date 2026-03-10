# Postgres as a Time-Series Database

Time-series data is everywhere: application metrics, IoT sensor readings, financial tick data, audit logs, user events, system telemetry. The conventional wisdom is that relational databases are bad at time series and you should reach for InfluxDB, TimescaleDB, or ClickHouse.

Partially true, partially not. Vanilla Postgres handles time-series workloads better than most people realize. TimescaleDB — which is Postgres — handles them even better. And for most applications generating moderate volumes of time-stamped data, the native Postgres approach is sufficient.

## What Makes Time-Series Data Different

Time-series data has characteristics that stress ordinary relational tables:

1. **Append-heavy writes:** New data arrives continuously; old data is rarely updated
2. **Time-range queries:** "Give me all readings between 9am and 10am yesterday"
3. **Recent data is hot:** Queries almost always touch recent data; old data is accessed infrequently
4. **Data volume:** Time-series data grows without bound; retention and downsampling are operational requirements
5. **Aggregation patterns:** "Average temperature per 5-minute bucket over the last week"

A plain `events` table with no special handling will work fine at small scale but degrade as it grows — indexes become large, queries scan too much data, and writes slow down.

The Postgres solutions:
- **Declarative partitioning** by time range for large tables
- **BRIN indexes** for efficient time-range queries
- **pg_partman** for automated partition management
- **Materialized views** for pre-aggregated rollups
- **TimescaleDB** for production-grade time-series at scale

## Native Postgres: Time-Range Partitioning

The most impactful optimization for time-series data in Postgres is range partitioning on the timestamp column.

```sql
-- Create a partitioned events table
CREATE TABLE metrics (
    id BIGINT NOT NULL,
    metric_name TEXT NOT NULL,
    value DOUBLE PRECISION NOT NULL,
    labels JSONB NOT NULL DEFAULT '{}',
    recorded_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (id, recorded_at)  -- partition key must be in PK
) PARTITION BY RANGE (recorded_at);

-- Create initial partitions (monthly)
CREATE TABLE metrics_2024_01 PARTITION OF metrics
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE metrics_2024_02 PARTITION OF metrics
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- ... etc.

-- Default partition catches anything outside defined ranges
CREATE TABLE metrics_default PARTITION OF metrics DEFAULT;
```

With range partitioning on `recorded_at`:
- Queries with time-range predicates benefit from *partition pruning* — Postgres only scans partitions that could contain matching rows
- Old data can be dropped instantly by dropping an entire partition (no DELETE, no VACUUM)
- Indexes on each partition are smaller and faster to scan

The partition granularity (daily, monthly, yearly) depends on your write volume and query patterns. Monthly partitions work well for most applications. Daily partitions work for very high write rates. Yearly partitions may be too coarse for retention purposes.

### BRIN Indexes on Partitions

Within each partition, a BRIN index on `recorded_at` is extremely efficient:

```sql
CREATE INDEX idx_metrics_2024_01_recorded_at
    ON metrics_2024_01 USING BRIN (recorded_at);
```

Since time-series data is naturally inserted in roughly chronological order, the physical order of rows in each partition correlates with `recorded_at`. BRIN's block-range min/max statistics are very accurate, allowing it to quickly identify which blocks to read.

For high-selectivity queries (short time ranges), you might prefer a B-tree index:
```sql
CREATE INDEX idx_metrics_2024_01_recorded_at_btree
    ON metrics_2024_01(recorded_at, metric_name);
```

The choice depends on your query patterns. BRIN is tiny and fast for range scans; B-tree is better for point queries and high-selectivity filters.

## Time-Bucket Aggregations

Time-series queries almost always involve grouping into time buckets:

```sql
-- Average metric value per 5-minute bucket
SELECT
    date_trunc('minute', recorded_at) -
        INTERVAL '1 minute' * (EXTRACT(MINUTE FROM recorded_at)::integer % 5) AS bucket,
    metric_name,
    avg(value) AS avg_value,
    min(value) AS min_value,
    max(value) AS max_value,
    count(*) AS sample_count
FROM metrics
WHERE metric_name = 'cpu_usage'
  AND recorded_at >= now() - interval '1 hour'
GROUP BY 1, 2
ORDER BY 1;
```

The time-bucket expression is verbose in vanilla Postgres. TimescaleDB provides `time_bucket()`:

```sql
-- With TimescaleDB:
SELECT
    time_bucket('5 minutes', recorded_at) AS bucket,
    metric_name,
    avg(value) AS avg_value
FROM metrics
WHERE recorded_at >= now() - interval '1 hour'
GROUP BY 1, 2
ORDER BY 1;
```

For vanilla Postgres, a helper function:

```sql
CREATE OR REPLACE FUNCTION time_bucket(bucket_size interval, ts timestamptz)
RETURNS timestamptz AS $$
    SELECT date_trunc('epoch',
        (EXTRACT(epoch FROM ts)::bigint / EXTRACT(epoch FROM bucket_size)::bigint)::bigint *
        EXTRACT(epoch FROM bucket_size)::bigint * interval '1 second' + 'epoch'::timestamptz)
$$ LANGUAGE SQL IMMUTABLE;
```

Or more simply for fixed intervals:

```sql
-- Hourly buckets
date_trunc('hour', recorded_at)

-- 15-minute buckets
date_trunc('hour', recorded_at) +
    INTERVAL '15 minutes' * floor(EXTRACT(MINUTE FROM recorded_at) / 15)
```

## Materialized Views for Rollups

Querying raw time-series data for aggregations over long time ranges is expensive. Pre-aggregating into rollup tables via materialized views dramatically improves query performance at the cost of some write overhead and refresh latency.

```sql
-- Hourly rollup materialized view
CREATE MATERIALIZED VIEW metrics_hourly AS
SELECT
    date_trunc('hour', recorded_at) AS hour,
    metric_name,
    avg(value) AS avg_value,
    min(value) AS min_value,
    max(value) AS max_value,
    percentile_cont(0.95) WITHIN GROUP (ORDER BY value) AS p95,
    count(*) AS sample_count
FROM metrics
WHERE recorded_at >= '2024-01-01'
GROUP BY 1, 2;

CREATE INDEX idx_metrics_hourly ON metrics_hourly(metric_name, hour DESC);

-- Refresh strategy: either on a schedule (via pg_cron) or incrementally
-- Full refresh:
REFRESH MATERIALIZED VIEW CONCURRENTLY metrics_hourly;
```

`REFRESH MATERIALIZED VIEW CONCURRENTLY` allows reads during refresh (it swaps in the new version atomically). Without `CONCURRENTLY`, the view is locked for the duration.

For incremental refresh (only updating recent hours), a `UNIQUE` index on the view is required for `CONCURRENTLY`:

```sql
CREATE UNIQUE INDEX idx_metrics_hourly_unique ON metrics_hourly(metric_name, hour);
```

### Hierarchical Rollups

A common pattern is multi-level rollups: raw → 5-minute → 1-hour → 1-day. Each level is computed from the previous:

```sql
-- 5-minute rollup from raw
CREATE MATERIALIZED VIEW metrics_5min AS
SELECT time_bucket('5 minutes', recorded_at) AS bucket, ...;

-- 1-hour rollup from 5-minute
CREATE MATERIALIZED VIEW metrics_1hour AS
SELECT date_trunc('hour', bucket) AS hour, ...
FROM metrics_5min
GROUP BY ...;
```

This avoids reprocessing raw data for higher-level aggregations.

## Data Retention

Time-series data needs retention policies. Keeping five years of second-resolution sensor data is expensive and usually unnecessary.

With partitioning, dropping old data is instant:

```sql
-- Drop everything older than 6 months
DROP TABLE IF EXISTS metrics_2024_01;  -- instant, no VACUUM needed
```

More carefully, using pg_partman (see Chapter 11):

```sql
-- Configure pg_partman retention
UPDATE partman.part_config
SET retention = '6 months',
    retention_keep_table = false
WHERE parent_table = 'public.metrics';
```

Without partitioning, a scheduled DELETE:

```sql
-- Run via pg_cron or a maintenance job
DELETE FROM metrics WHERE recorded_at < now() - interval '6 months';
```

DELETE on large tables is slow and generates dead tuples that must be vacuumed. Partitioning is strongly preferred for high-volume time-series data.

## TimescaleDB: Postgres for Time-Series at Scale

TimescaleDB is a PostgreSQL extension (from Timescale, Inc.) that adds time-series capabilities to Postgres. It's not a separate database — it's Postgres with additional time-series features.

**Core concept: Hypertables**

TimescaleDB abstracts partitioned tables as *hypertables* — regular-looking tables that are automatically partitioned into *chunks* under the hood:

```sql
-- Enable TimescaleDB
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Create a regular table
CREATE TABLE metrics (
    recorded_at TIMESTAMPTZ NOT NULL,
    metric_name TEXT NOT NULL,
    value DOUBLE PRECISION NOT NULL,
    labels JSONB NOT NULL DEFAULT '{}'
);

-- Convert it to a hypertable, partitioned by 1-day chunks
SELECT create_hypertable('metrics', 'recorded_at', chunk_time_interval => INTERVAL '1 day');
```

TimescaleDB handles:
- Automatic chunk creation as time advances
- Chunk pruning for time-range queries
- Per-chunk indexes (small, fast)
- Retention policies and chunk deletion
- Compression of old chunks

**Continuous Aggregates**

TimescaleDB's most powerful feature is *continuous aggregates* — materialized views that are incrementally refreshed automatically:

```sql
CREATE MATERIALIZED VIEW metrics_1hour
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', recorded_at) AS hour,
    metric_name,
    avg(value) AS avg_value,
    min(value) AS min_value,
    max(value) AS max_value,
    count(*) AS sample_count
FROM metrics
GROUP BY 1, 2;

-- Policy: refresh continuously, covering new data
SELECT add_continuous_aggregate_policy('metrics_1hour',
    start_offset => INTERVAL '3 hours',
    end_offset   => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

Unlike standard Postgres materialized views, continuous aggregates update incrementally — only processing new data since the last refresh, not the entire dataset. This makes them practical for real-time dashboards.

**Compression**

TimescaleDB can compress old chunks to dramatically reduce storage:

```sql
-- Enable compression on chunks older than 7 days
ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'metric_name'
);

SELECT add_compression_policy('metrics', INTERVAL '7 days');
```

Compressed chunks typically achieve 10–20x compression ratios for time-series data. They remain queryable but are read-only and slower to scan (decompression happens on read). TimescaleDB is smart about querying mixed compressed/uncompressed chunks.

**Retention Policies**

```sql
-- Automatically drop chunks older than 90 days
SELECT add_retention_policy('metrics', INTERVAL '90 days');
```

**Write Performance**

TimescaleDB is designed for high-ingest workloads. Tuning recommendations:

- Increase `max_wal_size` for batch inserts
- Set `timescaledb.max_background_workers` appropriately
- Use `COPY` for bulk loads
- Consider `synchronous_commit = off` for non-critical time-series (with understood trade-offs)

**When to use TimescaleDB vs. vanilla Postgres:**

Use TimescaleDB when:
- You're storing millions of data points per day
- You need automatic chunk management (don't want to manage partitions manually)
- Continuous aggregates are a core requirement
- Compression is important for storage costs
- You want time-series-specific query optimizations

Use vanilla Postgres when:
- Your time-series volume is modest (tens of thousands of rows per day)
- You want to minimize extensions and complexity
- You're already using partitioning via pg_partman
- TimescaleDB's licensing model (the Community vs. Enterprise distinction) is a concern

## Window Functions for Time-Series Analysis

Postgres's window functions are powerful for time-series analysis:

```sql
-- Running average (7-period moving average)
SELECT
    recorded_at,
    metric_name,
    value,
    avg(value) OVER (
        PARTITION BY metric_name
        ORDER BY recorded_at
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7
FROM metrics
WHERE recorded_at >= now() - interval '1 day'
ORDER BY metric_name, recorded_at;

-- Compare to previous period
SELECT
    date_trunc('hour', recorded_at) AS hour,
    metric_name,
    avg(value) AS current_avg,
    lag(avg(value)) OVER (
        PARTITION BY metric_name
        ORDER BY date_trunc('hour', recorded_at)
    ) AS prev_hour_avg
FROM metrics
WHERE recorded_at >= now() - interval '2 days'
GROUP BY 1, 2
ORDER BY 1, 2;

-- Cumulative sum
SELECT
    recorded_at,
    metric_name,
    value,
    sum(value) OVER (
        PARTITION BY metric_name
        ORDER BY recorded_at
        ROWS UNBOUNDED PRECEDING
    ) AS cumulative_sum
FROM metrics
ORDER BY metric_name, recorded_at;
```

## Gaps and Islands: Detecting Missing Data

A common time-series challenge: detecting gaps in data (missing sensor readings, outages):

```sql
-- Find gaps larger than 5 minutes in metric data
WITH ordered AS (
    SELECT
        recorded_at,
        lead(recorded_at) OVER (PARTITION BY metric_name ORDER BY recorded_at) AS next_recorded_at
    FROM metrics
    WHERE metric_name = 'temperature'
      AND recorded_at >= now() - interval '24 hours'
),
gaps AS (
    SELECT
        recorded_at AS gap_start,
        next_recorded_at AS gap_end,
        next_recorded_at - recorded_at AS gap_duration
    FROM ordered
    WHERE next_recorded_at - recorded_at > interval '5 minutes'
)
SELECT * FROM gaps ORDER BY gap_start;
```

## Postgres vs. InfluxDB: The Honest Comparison

**Postgres/TimescaleDB wins when:**
- You need SQL for time-series analysis (InfluxQL/Flux are less powerful)
- You need to join time-series with relational data
- You want one database to operate and understand
- Your scale is below ~100M data points per day per node
- You need ACID guarantees on time-series writes
- Operational simplicity is a priority

**InfluxDB/specialized time-series databases win when:**
- You're ingesting billions of events per day across multiple nodes
- You need horizontal write scaling
- Your team's workflow is built around Grafana/InfluxDB tooling
- You need line protocol for high-throughput IoT ingestion
- Your data is exclusively time-series with no relational requirements

**The honest truth:** TimescaleDB has been benchmarked at millions of data points per second on a single node. The threshold at which you need InfluxDB is genuinely high. For application metrics, user events, financial data, and IoT telemetry at typical startup or midsize company scale, TimescaleDB is an excellent choice. And since it's Postgres, you get SQL, JOINs, window functions, and all the operational benefits of the Postgres ecosystem.

The architecture story also matters: InfluxDB requires a separate database to operate and synchronize. TimescaleDB is your Postgres instance — one backup strategy, one monitoring setup, one mental model.

Time-series data is one of the clearer "Postgres is enough" cases. The specialized time-series databases solve a real problem, but that problem is usually at a scale that most teams don't face. Start with partitioned tables or TimescaleDB, and re-evaluate when you actually hit the limits.
