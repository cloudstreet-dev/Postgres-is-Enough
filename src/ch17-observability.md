# Observability

You can't improve what you can't measure. Postgres ships with a remarkably complete observability system built in — the `pg_stat_*` family of views, `EXPLAIN ANALYZE`, slow query logging, wait event tracking, and more. This chapter is about using those instruments effectively.

The goal of database observability is to be able to answer: "What is the database doing right now? What has it been doing? What's slow? Why?" Without good observability, you're flying blind. With it, performance investigations that used to take days take minutes.

## The `pg_stat_*` Views

Postgres maintains statistics about nearly everything it does, surfaced as views in the `pg_catalog` schema. These views are essential for understanding what your database is doing.

### `pg_stat_activity`

The most important view for real-time diagnostics. Shows all active connections and what they're doing.

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    left(query, 100) AS query_snippet,
    now() - query_start AS duration,
    now() - state_change AS state_duration
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC NULLS LAST;
```

The `state` column:
- `active`: Actively executing a query
- `idle`: Waiting for a query from the client
- `idle in transaction`: In a transaction, waiting between statements
- `idle in transaction (aborted)`: In a failed transaction (must be rolled back)
- `fastpath function call`: Executing a fast-path function

**`idle in transaction` is a red flag.** A connection that's `idle in transaction` holds locks for the duration. If a long transaction is idle (application code is doing slow work between database calls), those locks block other queries. Alert on connections that are `idle in transaction` for more than 30 seconds.

The `wait_event_type` and `wait_event` columns tell you what the backend is waiting for:

Common wait events:
- `Lock / relation`: Waiting for a table lock
- `Lock / tuple`: Waiting for a row lock
- `LWLock / buffer_content`: Waiting for a shared buffer lock
- `IO / DataFileRead`: Waiting for a page read from disk (cache miss)
- `Client / ClientRead`: Waiting for client to send the next query
- `IPC / BgWorkerShutdown`: Internal — background worker coordination

### `pg_stat_user_tables`

Per-table statistics:

```sql
SELECT
    relname,
    seq_scan,
    idx_scan,
    n_live_tup,
    n_dead_tup,
    round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS bloat_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;
```

High `seq_scan` relative to `idx_scan` suggests missing or unused indexes. High `n_dead_tup` suggests autovacuum isn't keeping up. `last_autovacuum` and `last_autoanalyze` tell you when maintenance last ran.

### `pg_stat_user_indexes`

Per-index usage statistics:

```sql
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

`idx_scan = 0` means this index has never been used since the last statistics reset. These are candidates for removal — they add write overhead and waste space without helping any query.

### `pg_stat_bgwriter`

Background writer and checkpoint statistics:

```sql
SELECT
    checkpoints_timed,
    checkpoints_req,
    round(checkpoints_req::numeric / nullif(checkpoints_timed + checkpoints_req, 0) * 100, 2) AS checkpoint_req_pct,
    buffers_checkpoint,
    buffers_clean,
    maxwritten_clean,
    buffers_alloc,
    stats_reset
FROM pg_stat_bgwriter;
```

`checkpoint_req_pct` high (> 20%) means checkpoints are being triggered by WAL size rather than time — increase `max_wal_size`. `maxwritten_clean > 0` means the background writer hit its limit — increase `bgwriter_lru_maxpages`.

### `pg_stat_replication`

Covered in Chapter 14. Essential for monitoring standby lag.

### `pg_stat_database`

Database-level aggregates:

```sql
SELECT
    datname,
    numbackends AS active_connections,
    xact_commit,
    xact_rollback,
    round(xact_rollback::numeric / nullif(xact_commit + xact_rollback, 0) * 100, 2) AS rollback_pct,
    blks_read,
    blks_hit,
    round(blks_hit::numeric / nullif(blks_read + blks_hit, 0) * 100, 2) AS cache_hit_pct,
    deadlocks,
    temp_files,
    pg_size_pretty(temp_bytes) AS temp_bytes_used
FROM pg_stat_database
WHERE datname = current_database();
```

`cache_hit_pct` below 95% suggests insufficient `shared_buffers` or a workload with poor cache locality. `temp_files` and `temp_bytes` nonzero indicates sorts and hash joins spilling to disk — increase `work_mem`. `deadlocks` counts should be near zero; any count warrants investigation.

## `pg_stat_statements`: Your Query Analysis Workhorse

Already covered in Chapter 11, but it bears repeating: `pg_stat_statements` is the single most important observability tool in Postgres.

```sql
-- Most time-consuming queries total
SELECT
    round(total_exec_time::numeric, 2) AS total_ms,
    calls,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    round(rows::numeric / calls, 1) AS avg_rows,
    left(query, 120) AS query
FROM pg_stat_statements
WHERE calls > 50
ORDER BY total_exec_time DESC
LIMIT 20;

-- High variance queries (sometimes fast, sometimes slow)
SELECT
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    round(stddev_exec_time::numeric / nullif(mean_exec_time, 0) * 100, 2) AS cv_pct,
    calls,
    left(query, 120) AS query
FROM pg_stat_statements
WHERE calls > 100
ORDER BY cv_pct DESC
LIMIT 20;

-- Queries with most cache misses (lots of disk reads)
SELECT
    round(shared_blks_read::numeric / calls, 1) AS avg_disk_reads,
    round(shared_blks_hit::numeric / calls, 1) AS avg_cache_hits,
    calls,
    left(query, 120) AS query
FROM pg_stat_statements
WHERE calls > 10
ORDER BY avg_disk_reads DESC
LIMIT 20;
```

Reset statistics to get a fresh baseline:
```sql
SELECT pg_stat_reset();              -- Reset all pg_stat_* views
SELECT pg_stat_statements_reset();   -- Reset pg_stat_statements only
```

## EXPLAIN ANALYZE: Query-Level Diagnostics

`EXPLAIN (ANALYZE, BUFFERS)` is the microscope for individual queries. Know how to read it.

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT u.id, u.email, count(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > now() - interval '30 days'
GROUP BY u.id, u.email
ORDER BY order_count DESC
LIMIT 20;
```

Example output breakdown:

```
Sort  (cost=14523.00..14523.05 rows=20 width=48) (actual time=234.1..234.1 rows=20 loops=1)
  Sort Key: (count(o.id)) DESC
  Sort Method: top-N heapsort  Memory: 26kB
  Buffers: shared hit=8234 read=126
  ->  HashAggregate  (cost=14500.00..14520.00 rows=2000 width=48) (actual time=231.4..234.0 rows=1523 loops=1)
        Group Key: u.id, u.email
        Batches: 1  Memory Usage: 369kB
        Buffers: shared hit=8234 read=126
        ->  Hash Left Join  (cost=... rows=...  (actual time=... loops=1)
              Hash Cond: (o.user_id = u.id)
              Buffers: shared hit=8234 read=126
              ->  Seq Scan on orders  (cost=0.00..5280.00 rows=... (actual time=... loops=1)
                    Buffers: shared hit=2100 read=80
              ->  Hash  (cost=... (actual time=... loops=1)
                    Buckets: 2048  Batches: 1  Memory Usage: 121kB
                    Buffers: shared hit=6134 read=46
                    ->  Index Scan using idx_users_created on users  (cost=... (actual time=...)
                          Index Cond: (created_at > ...)
                          Buffers: shared hit=6134 read=46
Planning Time: 0.8 ms
Execution Time: 234.2 ms
```

Key patterns to recognize:

**Seq Scan on a large table:** The query is reading the whole table. Look for missing indexes.

**Nested Loop with large outer:**
```
Nested Loop  (cost=... rows=100000)
  ->  Seq Scan on large_table  (rows=100000)
  ->  Index Scan on other_table  (rows=1)
```
This is `100000 × 1` index lookups. Fine if the outer is small; expensive if large.

**Rows estimate vs. actual significantly different:**
```
Seq Scan on orders  (cost=... rows=100 ...) (actual ... rows=95000 ...)
```
The planner thought there'd be 100 rows but found 95,000 — bad statistics. Run `ANALYZE orders`.

**`Buffers: read` is high:** Pages being read from disk, not buffer cache. Either the data isn't hot in the cache, or `shared_buffers` is too small for this workload.

**`Sort Method: external merge`:** Sort spilled to disk. Increase `work_mem` for this type of query.

## Slow Query Logging

Configure Postgres to log queries that exceed a duration threshold:

```ini
log_min_duration_statement = 1000  # Log queries > 1 second
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

This generates log entries for slow queries, including the full query text, duration, and context. Feed these logs to pgBadger for analysis.

**Careful with `log_statement = 'all'`:** This logs every statement and generates enormous log volumes on a busy system. Use `log_min_duration_statement` instead.

## pgBadger: Log Analysis

pgBadger is a Perl script that analyzes Postgres log files and generates HTML reports showing:
- Slowest queries (sorted by count, total time, average time)
- Most frequent queries
- Queries per second over time
- Connections, errors, and lock waits over time
- Wait event statistics

```bash
pgbadger /var/log/postgresql/postgresql.log -o report.html
```

Run pgBadger daily on your log files and review the report. The "top 10 slowest queries by total time" section is where you start performance investigations.

## Prometheus Integration: `postgres_exporter`

For infrastructure-level metrics and alerting, `postgres_exporter` (from Prometheus community) exposes Postgres metrics in Prometheus format:

```yaml
# docker-compose.yml
services:
  postgres_exporter:
    image: prometheuscommunity/postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://exporter:password@postgres:5432/mydb?sslmode=disable"
    ports:
      - "9187:9187"
```

Key metrics exported:
- `pg_stat_database_blks_hit` / `pg_stat_database_blks_read` (cache hit ratio)
- `pg_stat_user_tables_n_dead_tup` (bloat)
- `pg_stat_replication_*` (replication lag)
- `pg_stat_bgwriter_*` (checkpoint and writer stats)
- `pg_locks_count` (lock contention)
- `pg_stat_activity_count` (active connections)

**Essential Grafana dashboards:**
- PostgreSQL Database (by Percona or official Postgres exporter)
- Dashboard ID 9628 or 12273 on grafana.com

Set up alerts for:
- Cache hit ratio < 95%
- Replication lag > 60 seconds
- Dead tuple percentage > 20% on any table
- Number of `idle in transaction` connections > 5
- Checkpoint requested count growing (WAL pressure)
- Total connections > 80% of `max_connections`

## Wait Event Analysis

When Postgres is slow and you're not sure why, wait events tell you where time is being spent:

```sql
SELECT wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE state = 'active'
GROUP BY 1, 2
ORDER BY count DESC;
```

What the common wait events mean:
- **`Lock/relation`**: Table-level lock contention — migrations, VACUUM FULL, or high DDL rate
- **`Lock/tuple`**: Row-level lock contention — hot rows being updated by many connections
- **`LWLock/buffer_content`**: Shared buffer access contention — many connections reading/writing the same buffer
- **`IO/DataFileRead`**: Disk reads — cache misses, large sequential scans
- **`IO/WALWrite`**: WAL write overhead — heavy write workload
- **`Client/ClientRead`**: Waiting for application — application is slow sending the next query

Wait events give you a diagnostic shortcut: instead of guessing why queries are slow, you can observe what they're blocked on and fix that specifically.

## The Observability Stack

A complete observability setup for a production Postgres instance:

```
Postgres logs → pgBadger → daily HTML report
Postgres metrics → postgres_exporter → Prometheus → Grafana dashboards
pg_stat_statements → periodic snapshot → trend analysis
pg_stat_activity → alerting → PagerDuty/Slack (on blocked queries, lock waits)
EXPLAIN ANALYZE → when investigating specific slow queries
```

This is not complex to set up, and the payoff is enormous. The difference between a team that's reactive to database performance issues and one that catches them early is almost always whether they have this observability stack in place.

The best observability investment: add `pg_stat_statements` to `shared_preload_libraries`, set up `postgres_exporter` with Prometheus and Grafana, and configure `log_min_duration_statement = 1000`. These three steps give you 80% of the visibility you need, and they can be done in an afternoon.
