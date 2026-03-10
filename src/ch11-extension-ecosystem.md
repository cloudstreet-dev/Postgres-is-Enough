# The Extension Ecosystem

Extensions are what make PostgreSQL a platform rather than just a database. The core Postgres installation is already excellent. The extension ecosystem transforms it into a system that handles geospatial data, fuzzy string matching, full-text with synonyms, time series, vector search, cryptographic operations, job scheduling, and much more — all without leaving Postgres.

This chapter covers the extensions worth knowing and installing, how extensions work under the hood, and how to evaluate whether an extension is worth the dependency.

## How Extensions Work

A PostgreSQL extension is a bundle of SQL objects (functions, types, operators, indexes, tables) and optional shared libraries (C code loaded into the Postgres process) that extend the database's capabilities.

To install an extension, you first make the extension files available to the Postgres installation (usually by installing a package like `postgresql-16-pgvector`), then enable it in a specific database:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

This runs the extension's SQL install script, creating the types, functions, and other objects in the current database. Extensions are per-database, not per-cluster.

To see installed extensions:
```sql
SELECT name, default_version, installed_version, comment
FROM pg_available_extensions
WHERE installed_version IS NOT NULL;
```

To see what objects an extension created:
```sql
SELECT classid::regclass, objid, deptype
FROM pg_depend
WHERE refobjid = (SELECT oid FROM pg_extension WHERE extname = 'pg_trgm');
```

Extensions that use C code load a shared library into the Postgres backend processes. These extensions can crash the backend if they have bugs — this is a risk worth understanding. Stick to well-maintained, widely-used extensions.

## The Essential Extensions

### `pg_stat_statements`

**What it does:** Tracks statistics on all SQL statements executed — query text, execution count, total/mean/min/max execution time, rows processed, buffer usage.

**Why you need it:** Without `pg_stat_statements`, finding your slow queries means trawling through logs. With it, you can directly query which queries are taking the most time.

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Must be loaded at startup (add to postgresql.conf):
-- shared_preload_libraries = 'pg_stat_statements'

-- Top 10 queries by total execution time:
SELECT
    round(total_exec_time::numeric, 2) AS total_ms,
    calls,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    left(query, 100) AS query_snippet
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Install this on every Postgres instance. No exceptions.

### `pg_trgm`

**What it does:** Trigram-based similarity functions and operators for fuzzy string matching, plus GIN/GiST index support for fast `LIKE`/`ILIKE` queries.

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);

-- Fast fuzzy search, handles typos
SELECT name FROM products WHERE name % 'postrges';

-- Fast LIKE query (normally requires full scan)
SELECT name FROM products WHERE name ILIKE '%laptop%';
```

Essential for any application with user-facing search. See Chapter 6.

### `pgcrypto`

**What it does:** Cryptographic functions — hashing, symmetric and asymmetric encryption, random bytes.

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Secure password hashing (bcrypt)
INSERT INTO users (email, password_hash)
VALUES ('alice@example.com', crypt('password123', gen_salt('bf', 12)));

-- Verify password
SELECT id FROM users
WHERE email = 'alice@example.com'
  AND password_hash = crypt('password123', password_hash);

-- Random UUID (also available as gen_random_uuid() without pgcrypto in PG 13+)
SELECT gen_random_uuid();

-- Cryptographically random bytes
SELECT encode(gen_random_bytes(32), 'hex');
```

Note: For password hashing, prefer bcrypt (`gen_salt('bf', cost)`) over md5 or sha variants. The cost factor (default 8, reasonable values 10-14) controls bcrypt's computational expense.

### `uuid-ossp`

**What it does:** Functions for generating various UUID versions.

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

SELECT uuid_generate_v4();  -- random UUID
SELECT uuid_generate_v1();  -- timestamp-based UUID (leaks MAC address — avoid)
```

As of PostgreSQL 13, `gen_random_uuid()` (v4) is built-in without an extension. For most purposes, you don't need `uuid-ossp`. Use `gen_random_uuid()` directly.

### `pg_partman`

**What it does:** Automated partition management for range and list partitions. Creates new partitions on schedule, maintains partition sets, enforces retention policies.

```sql
CREATE EXTENSION IF NOT EXISTS pg_partman;

-- Register a partitioned table with pg_partman
SELECT partman.create_parent(
    p_parent_table => 'public.metrics',
    p_control => 'recorded_at',
    p_type => 'native',
    p_interval => '1 month',
    p_premake => 3  -- Create 3 future partitions in advance
);

-- Configure retention
UPDATE partman.part_config
SET retention = '12 months',
    retention_keep_table = false,
    infinite_time_partitions = true
WHERE parent_table = 'public.metrics';

-- Run maintenance (typically via pg_cron)
SELECT partman.run_maintenance();
```

`pg_partman` eliminates the operational burden of manually creating monthly partitions. Instead of a cron job that creates next month's partition, pg_partman handles it automatically via a background maintenance function.

Use `pg_partman` for any time-partitioned table that grows indefinitely.

### `pg_cron`

**What it does:** A cron-style scheduler built into Postgres. Runs SQL statements or stored procedures on a schedule.

```sql
-- Must be added to shared_preload_libraries first:
-- shared_preload_libraries = 'pg_cron'

CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Clean up expired sessions every hour
SELECT cron.schedule(
    'cleanup-sessions',
    '0 * * * *',  -- cron expression: every hour at :00
    $$DELETE FROM sessions WHERE expires_at < now()$$
);

-- Run partition maintenance daily at midnight
SELECT cron.schedule(
    'partition-maintenance',
    '0 0 * * *',
    $$SELECT partman.run_maintenance()$$
);

-- Refresh a materialized view every 5 minutes
SELECT cron.schedule(
    'refresh-metrics-view',
    '*/5 * * * *',
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY metrics_hourly$$
);

-- List scheduled jobs
SELECT jobid, jobname, schedule, command, active
FROM cron.job;

-- View job execution history
SELECT jobid, status, return_message, start_time, end_time
FROM cron.job_run_details
ORDER BY start_time DESC
LIMIT 20;
```

`pg_cron` eliminates cron entries on the host machine for database maintenance tasks. Everything lives in Postgres, visible and auditable from SQL.

### `PostGIS`

**What it does:** The gold standard for geospatial data in any database. Adds geometry and geography types, hundreds of spatial functions, and GiST indexes for spatial queries.

```sql
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE locations (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    location GEOGRAPHY(POINT, 4326)  -- WGS84 lat/lon
);

CREATE INDEX idx_locations_gist ON locations USING GIST (location);

-- Insert a point (longitude, latitude order for GeoJSON compatibility)
INSERT INTO locations (name, location)
VALUES ('Empire State Building', ST_MakePoint(-73.9857, 40.7484));

-- Find locations within 1 km of a point
SELECT name, ST_Distance(location, ST_MakePoint(-73.9857, 40.7484)::geography) AS distance_m
FROM locations
WHERE ST_DWithin(location, ST_MakePoint(-73.9857, 40.7484)::geography, 1000)
ORDER BY distance_m;
```

PostGIS is mature, extremely capable, and essential for any geospatial application. If your application uses geospatial data and you're not using PostGIS, you're doing it wrong.

### `pgvector`

Covered extensively in Chapter 9. The extension for vector similarity search. Essential for any AI-powered feature.

### `timescaledb`

Covered extensively in Chapter 10. The extension for production-grade time-series data management.

### `pg_repack`

**What it does:** Online table and index reorganization — removes bloat, reclaims space, reorders rows — without locking. `VACUUM FULL` is the standard approach but locks the table. `pg_repack` does it concurrently.

```sql
-- pg_repack is run as a command-line tool, not a SQL function
-- pg_repack --table orders mydb

-- Or via pgRepack function (after installation):
SELECT repack.repack_table('orders');
```

Use `pg_repack` when a table has severe bloat that `VACUUM` can't reclaim (e.g., after massive bulk deletes) and you can't afford `VACUUM FULL`'s table lock.

### `pgtap`

**What it does:** A unit testing framework for Postgres. Write tests in SQL.

```sql
CREATE EXTENSION IF NOT EXISTS pgtap;

-- Test that users.email is unique
SELECT plan(2);
SELECT has_unique('public', 'users', ARRAY['email'], 'users.email is unique');
SELECT col_not_null('public', 'users', 'email', 'users.email is not null');
SELECT finish();
```

Useful for CI pipelines that need to verify schema constraints, function behavior, and data integrity.

### `pg_hint_plan`

**What it does:** Allows query plan hints — telling the query planner which index to use, which join strategy to apply, etc. Similar to Oracle's hint system.

```sql
CREATE EXTENSION IF NOT EXISTS pg_hint_plan;

-- Hint: use the specific index
/*+ IndexScan(users idx_users_email) */
SELECT * FROM users WHERE email = 'alice@example.com';

-- Hint: prefer nested loop join
/*+ NestLoop(orders users) */
SELECT * FROM orders JOIN users ON users.id = orders.user_id WHERE orders.id = 1;
```

`pg_hint_plan` is a debugging and emergency tool, not a crutch. If you find yourself needing hints regularly, the underlying problem is bad statistics, a missing index, or a schema design issue. Fix the root cause. Use hints as a temporary workaround when you're debugging a production issue.

### `pgaudit`

**What it does:** Detailed audit logging of database operations — which user ran which query, when, on which objects.

```sql
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- Configure via postgresql.conf:
-- pgaudit.log = 'write, ddl'  -- Log all writes and DDL
-- pgaudit.log_relation = on    -- Log relation name in log

-- Or per-role:
ALTER ROLE audited_user SET pgaudit.log = 'all';
```

Essential for compliance workloads (PCI, HIPAA, SOC 2) that require detailed audit trails. See Chapter 16.

### `plpgsql` (built-in) and Other Languages

PL/pgSQL is built-in and is the standard language for Postgres functions and triggers. Other languages are available as extensions:

- **`plpython3u`:** Write functions in Python (untrusted — can access the filesystem)
- **`plv8`:** Write functions in JavaScript (V8 engine)
- **`plrust`:** Write functions in Rust (safe, high-performance)
- **`plperl`:** Write functions in Perl

Most applications don't need server-side code beyond PL/pgSQL. PL/pgSQL handles complex triggers, functions, and procedures well. Reach for other languages only when PL/pgSQL is genuinely limiting.

## Evaluating an Extension

Before adding an extension to production, ask:

1. **Is it actively maintained?** Check the GitHub repo. Are there recent commits? Open issues being responded to? An unmaintained extension is a liability for every Postgres major version upgrade.

2. **Does it require `shared_preload_libraries`?** Extensions loaded at startup (`pg_stat_statements`, `pg_cron`, `timescaledb`, `pgaudit`) require a server restart to add or remove. Plan for this.

3. **Is it compiled C code or pure SQL?** C extensions can crash the backend if they have bugs. Pure SQL extensions have no such risk. Prefer pure SQL or well-audited C extensions.

4. **What happens when it's dropped?** `DROP EXTENSION` removes the SQL objects. Any data stored in extension tables is lost. Any columns using extension types would need to be migrated first.

5. **Will it survive a major version upgrade?** Extensions must be upgraded alongside Postgres. Some have better upgrade paths than others. Check the extension's documentation for upgrade instructions.

6. **Does the cloud provider support it?** AWS RDS, Google Cloud SQL, Azure Database for PostgreSQL, and Supabase have curated extension lists. Some extensions require superuser privileges that managed providers don't grant. Verify your target environment before committing.

## Extensions and Managed Postgres

Most managed Postgres services support a subset of extensions:

- **AWS RDS/Aurora:** Supports most common extensions. `pg_stat_statements`, `pg_trgm`, `pgcrypto`, `pgvector`, `PostGIS`, `pg_partman` — yes. `pg_cron` — yes, as of recent versions. `timescaledb` — no (use Timescale Cloud or self-hosted).

- **Supabase:** Excellent extension support. Most extensions in this chapter are available with one click.

- **Google Cloud SQL:** Good extension support. `pg_stat_statements`, `pg_trgm`, `pgvector`, `PostGIS`, others.

- **Neon:** Growing extension support, focused on serverless Postgres.

Always verify extension availability on your target platform before architecture decisions that depend on them.

## The Right Philosophy

Extensions are a superpower, but they're also dependencies. Each extension you add is a library your Postgres installation depends on — for upgrades, for security patches, for operational support. The extensions in this chapter are all mature and widely used. They're worth the dependency.

The test for a new extension: is it solving a real problem, is it actively maintained, and is its maintenance burden less than the alternative (a separate database, a different approach)? Most of the extensions in this chapter pass this test decisively. For others, evaluate carefully before committing to them in production.

The core insight: Postgres's extension system is what makes "Postgres is enough" possible at all. The base database is excellent. The extensions make it comprehensive.
