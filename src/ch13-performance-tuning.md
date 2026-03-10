# Performance Tuning

A default Postgres installation is configured conservatively — appropriate for a development machine or a small VM, but leaving significant performance on the table for a production server. This chapter covers the configuration parameters that matter, the autovacuum behavior you need to understand, connection pooling as a force multiplier, and the query optimization habits that prevent most performance problems.

Postgres performance tuning is iterative. There is no single configuration change that makes everything fast. The process is: measure, identify bottlenecks, tune, measure again.

## Memory Configuration

### `shared_buffers`

The most important memory setting. Controls the size of Postgres's shared buffer cache — the pool of memory that all backends share for caching data pages.

**Default:** 128MB (egregiously low for production)

**Recommendation:** ~25% of available RAM

```ini
shared_buffers = 8GB  # on a 32GB server
```

Setting `shared_buffers` higher than 25% of RAM has diminishing returns because the OS page cache also caches frequently-read data. Going above 40% of RAM can actually hurt performance by reducing the OS page cache.

### `effective_cache_size`

Not a memory allocation — it's a hint to the query planner about how much total memory (shared_buffers + OS cache) is available for caching. The planner uses this to decide between index scans and sequential scans.

**Recommendation:** 50–75% of total RAM

```ini
effective_cache_size = 24GB  # on a 32GB server
```

If this is set too low, the planner incorrectly thinks the disk is slow and prefers sequential scans when it should use indexes.

### `work_mem`

Memory available per sort or hash operation. Each sort, hash join, and hash aggregate can use up to `work_mem`. A query with multiple operations can use `work_mem` multiple times. With many concurrent connections, total memory usage can be `work_mem × connections × operations_per_query`.

**Default:** 4MB (too low for complex queries)

**Recommendation:** Balance between query complexity and connection count. A formula:
```
work_mem = (RAM - shared_buffers) / (max_connections × 2)
```

For a 32GB server with 8GB shared_buffers and 100 max_connections:
```
work_mem = (32 - 8) / (100 × 2) ≈ 120MB
```

Setting `work_mem` too high with many connections causes OOM. Set conservatively globally and increase per-session for complex analytical queries:

```sql
SET work_mem = '256MB';
SELECT ... ORDER BY ... LIMIT ...;  -- benefits from higher work_mem
RESET work_mem;
```

Watch for "external sort" operations in `EXPLAIN` output — they indicate the sort spilled to disk because `work_mem` was too low.

### `maintenance_work_mem`

Memory for maintenance operations: `VACUUM`, `CREATE INDEX`, `ALTER TABLE`, etc.

**Recommendation:** 1GB or more for systems with large tables. This speeds up index creation dramatically.

```ini
maintenance_work_mem = 2GB
```

### `max_wal_size`

Controls when checkpoint processing forces a WAL flush. Too small causes frequent checkpoints, generating I/O spikes. Too large increases recovery time after a crash.

**Recommendation:** 2–4GB for most systems, higher for write-heavy workloads.

```ini
max_wal_size = 4GB
```

Watch `pg_stat_bgwriter.checkpoint_req` — if this is high, checkpoints are being triggered by WAL size rather than time. Increase `max_wal_size`.

## Checkpoint Configuration

Checkpoints flush dirty pages from the buffer cache to disk. A checkpoint that runs too fast causes I/O spikes (all the dirty pages written in a burst). A checkpoint that runs too slow causes long recovery times after a crash.

```ini
checkpoint_completion_target = 0.9  # Spread I/O over 90% of checkpoint interval
checkpoint_timeout = 15min           # Maximum time between checkpoints
```

`checkpoint_completion_target = 0.9` (the default in recent Postgres versions) is good. It tells Postgres to spread checkpoint writes over 90% of the interval between checkpoints, smoothing I/O.

## WAL Settings

```ini
wal_level = replica         # Minimum for streaming replication
wal_compression = on        # Compress WAL records (reduces WAL volume)
wal_buffers = 16MB          # WAL write buffer (usually auto-configured from shared_buffers)
synchronous_commit = on     # Don't change this unless you understand the trade-off
```

`synchronous_commit = off` gives a performance boost (no fsync on commit) at the cost of potentially losing the last few seconds of committed transactions on crash. This is acceptable for non-critical data (analytics events, logs, rate limit counters). Never set it off globally for transactional data.

## Autovacuum: The Most Misunderstood Setting

Autovacuum is the background process that reclaims dead tuple space, updates table statistics, and prevents XID wraparound. It is not optional. Disabling autovacuum is not a performance optimization — it's setting up a future disaster.

The most important autovacuum settings:

### `autovacuum_vacuum_scale_factor` and `autovacuum_vacuum_threshold`

These control when autovacuum triggers on a table. The formula:
```
threshold = autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * reltuples
```

**Defaults:**
- `autovacuum_vacuum_threshold = 50` (absolute minimum dead tuples)
- `autovacuum_vacuum_scale_factor = 0.2` (20% of table)

For a table with 10 million rows, autovacuum triggers when there are 2,000,050 dead tuples. For a frequently-updated table, this means autovacuum runs rarely, and dead tuples accumulate heavily before cleanup.

**Recommendation for large tables:** Lower the scale factor significantly, or set per-table thresholds:

```sql
-- Per-table: vacuum after 1% dead tuples (instead of 20%)
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_scale_factor = 0.005,
    autovacuum_analyze_threshold = 500
);
```

### `autovacuum_max_workers`

**Default:** 3. The number of autovacuum workers that can run simultaneously. For a system with many large, frequently-updated tables, the default is often insufficient.

```ini
autovacuum_max_workers = 6
```

More workers means more CPU and I/O, but tables stay cleaner and queries stay fast.

### `autovacuum_cost_delay` and `autovacuum_vacuum_cost_limit`

Autovacuum throttles itself using a cost-based mechanism to avoid overwhelming I/O. Each page read, dirty page write, and dirty page hit has a cost. When the accumulated cost reaches `autovacuum_vacuum_cost_limit`, autovacuum sleeps for `autovacuum_cost_delay` milliseconds.

**Defaults:**
- `autovacuum_cost_delay = 2ms` (recent versions)
- `autovacuum_vacuum_cost_limit = 200`

For SSDs, autovacuum can be much more aggressive. The default throttling is designed for spinning disks:

```ini
# For NVMe SSDs:
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 2000  # 10x more aggressive
```

### Monitoring Autovacuum

```sql
-- See when tables were last vacuumed and their bloat
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Tables approaching autovacuum threshold
SELECT
    schemaname,
    relname,
    n_dead_tup,
    n_live_tup,
    autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * n_live_tup AS vacuum_threshold
FROM pg_stat_user_tables
JOIN pg_class ON relname = pg_class.relname
LEFT JOIN (
    SELECT relid,
        (array_to_string(reloptions, ',') ~ 'autovacuum_vacuum_threshold')::boolean AS has_threshold,
        (regexp_match(array_to_string(reloptions, ','), 'autovacuum_vacuum_threshold=(\d+)'))[1]::bigint AS autovacuum_vacuum_threshold,
        (regexp_match(array_to_string(reloptions, ','), 'autovacuum_vacuum_scale_factor=(\d+\.?\d*)'))[1]::numeric AS autovacuum_vacuum_scale_factor
    FROM pg_class
) opts ON opts.relid = pg_class.oid
CROSS JOIN (
    SELECT current_setting('autovacuum_vacuum_threshold')::bigint AS autovacuum_vacuum_threshold,
           current_setting('autovacuum_vacuum_scale_factor')::numeric AS autovacuum_vacuum_scale_factor
) defaults;
```

## Connection Pooling with PgBouncer

Postgres connections are expensive: each backend process uses ~5-10MB of RAM and involves OS process creation overhead. Applications that open many short-lived connections — serverless functions, high-concurrency APIs — can overwhelm Postgres's connection capacity.

PgBouncer is a lightweight connection pooler that sits between your application and Postgres, multiplexing many application connections onto a smaller number of Postgres connections.

### Pool Modes

**Session mode:** A server connection is assigned to a client for the duration of the client session. No query-level multiplexing. Only useful for reducing connection overhead (not connection count).

**Transaction mode:** A server connection is assigned for the duration of each transaction. After the transaction commits or rolls back, the connection returns to the pool. This is the most useful mode — a pool of 50 server connections can handle thousands of concurrent application connections.

**Statement mode:** A server connection is assigned for a single statement, then released. Most restrictive — prepared statements, `SET` commands, and transactions spanning multiple statements don't work.

For most applications, **transaction mode** is the right choice.

### PgBouncer Configuration

```ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr = 127.0.0.1
listen_port = 6432

pool_mode = transaction

# Server connections
max_client_conn = 10000    # Allow many application connections
default_pool_size = 25     # Postgres sees max 25 connections from this app
min_pool_size = 5
reserve_pool_size = 5

# Authentication
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Timeouts
server_idle_timeout = 600
client_idle_timeout = 0
query_timeout = 0

# Logging
log_connections = 0
log_disconnections = 0
```

**What doesn't work in transaction mode:** `SET LOCAL` (within a transaction is fine), session-level `SET`, advisory locks held across transactions, `LISTEN/NOTIFY`, server-side cursors outside transactions, prepared statements (unless you use `server_reset_query` or PgBouncer's prepared statement tracking feature). Know these limitations before adopting.

### How Many Postgres Connections?

The optimal number of Postgres server connections for throughput:

```
optimal_connections ≈ CPU_count * 2 + effective_spindle_count
```

For a 4-core server with an NVMe SSD:
```
optimal_connections ≈ 4 * 2 + 1 ≈ 9
```

This seems shockingly low but is supported by benchmarks. More connections than CPUs means context-switching overhead and lock contention outweigh concurrency benefits. For most servers, 20-50 Postgres connections is sufficient for high throughput. PgBouncer's job is to make 1000 application clients share those efficiently.

## `max_connections`

**Default:** 100. Set this based on what you can afford in terms of memory, not what you hope to use.

Each connection uses about 5-10MB of RAM. For a server with 32GB RAM and 8GB `shared_buffers`, the remaining 24GB can support roughly 2,400-4,800 connections. But you don't want that many — use PgBouncer instead and keep `max_connections` low.

```ini
max_connections = 200  # With PgBouncer handling the fan-out
```

## Query Performance: The EXPLAIN Habit

No amount of server configuration compensates for missing indexes or bad queries. The most impactful performance work is query-level.

The diagnostic workflow:

1. **Find slow queries** via `pg_stat_statements`:
```sql
SELECT query, calls, total_exec_time / calls AS avg_ms, rows / calls AS avg_rows
FROM pg_stat_statements
WHERE calls > 100
ORDER BY avg_ms DESC
LIMIT 20;
```

2. **Explain the slow query:**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.created_at > now() - interval '7 days'
  AND o.status = 'pending';
```

3. **Look for:**
   - Sequential scans on large tables → need an index
   - Nested loop with large outer relation → bad join choice, likely bad statistics
   - High `Buffers: read` → cache miss, data not in buffer
   - `rows=X (actual rows=Y)` with large discrepancy → bad statistics, run `ANALYZE`
   - Sort operations with "external sort" → increase `work_mem`
   - Filter: `(Rows Removed by Filter: N)` with large N → index doesn't exist or isn't selective enough

4. **Fix the issue** (add index, update statistics, rewrite query)

5. **Verify improvement** (run `EXPLAIN ANALYZE` again)

### Statistics and `ANALYZE`

Run `ANALYZE` after large bulk loads to update table statistics before the planner has to guess:

```sql
ANALYZE orders;  -- Update statistics for one table
ANALYZE;         -- Update statistics for all tables in current database
```

For columns with high cardinality or very uneven distributions, increase per-column statistics:

```sql
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;
```

### Parallel Query

Postgres can parallelize query execution across multiple CPU cores for sequential scans and some aggregations. This is controlled by `max_parallel_workers_per_gather`.

```ini
max_parallel_workers_per_gather = 4   # Up to 4 workers per parallel query
max_parallel_workers = 8              # Total parallel workers (across all queries)
max_worker_processes = 16             # Total background workers
```

Parallel query is automatic — the planner decides when to use it. It helps large analytical queries; it doesn't help indexed lookups.

## Partitioning for Performance

As covered in Chapter 3, time-partitioned tables with recent-data access patterns benefit enormously from partition pruning:

```sql
-- This only scans the relevant monthly partition:
SELECT * FROM events
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01';
```

For queries that always filter on the partition key, partitioning is a very effective performance strategy for large tables.

## Identifying Bloat

Index and table bloat degrades performance and wastes disk space. A quick check:

```sql
-- Find tables with significant dead tuple bloat
SELECT schemaname, relname, n_dead_tup,
       pg_size_pretty(pg_total_relation_size(schemaname || '.' || relname)) AS total_size
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC
LIMIT 20;

-- pgstattuple for precise bloat measurement (requires extension)
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstattuple('orders');
```

For severe bloat that `autovacuum` can't reclaim (e.g., after a massive delete), consider `VACUUM FULL` (locks the table) or `pg_repack` (online, no locking).

## Configuration Validation Tools

**pgtune** (pgtune.leopard.in.ua) generates a `postgresql.conf` based on your hardware profile. It's a good starting point.

**pgBadger** analyzes Postgres log files and produces detailed reports on slow queries, wait events, and error patterns.

**check_postgres** is a Nagios/Icinga monitoring plugin that checks connection counts, bloat, vacuum age, and many other indicators.

The key principle: Postgres's default configuration is a safe minimum, not a target. Every production Postgres instance should be tuned for its workload and hardware. The changes in this chapter — higher `shared_buffers`, lower autovacuum scale factors, PgBouncer in transaction mode — produce immediate, measurable improvements on almost every system.
