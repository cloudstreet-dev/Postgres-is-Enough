# The Stack — Putting It All Together

Twenty chapters later, here's what we know: Postgres is a relational database that can also store documents, search text, queue jobs, serve as a key-value store, index vectors, handle time-series data, and do most of what five specialized databases would otherwise do. It's ACID-compliant, well-understood, battle-tested, and backed by thirty years of engineering. And it doesn't require you to run six separate systems to do it.

This final chapter ties everything together into a reference architecture — the stack that a well-run, Postgres-centric team actually runs.

## The Core Principle

Design your stack around your actual requirements, not theoretical scale. Start simple. Add components only when you have evidence that the simpler approach is insufficient. Every component you add is a component you operate, monitor, debug, and eventually upgrade forever.

The reference architecture in this chapter is for a typical production web application or API — something serving thousands to millions of users. The vast majority of products fall into this category. Scale up from here when evidence demands it.

## The Reference Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Application Layer                       │
│                  (your web app / API servers)                  │
└──────────────────────────┬──────────────────────────────────┘
                           │ connection pool
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                        PgBouncer                               │
│                   (transaction mode pooler)                    │
└──────────────────────────┬──────────────────────────────────┘
                           │ 20–50 Postgres connections
            ┌──────────────┴──────────────┐
            ▼                             ▼
┌───────────────────┐          ┌───────────────────┐
│  Postgres Primary  │◄────────│  Postgres Standby │
│  (read/write)      │ stream  │  (read-only)       │
└────────────┬──────┘ repl.   └───────────────────┘
             │
             │ WAL archiving
             ▼
┌───────────────────┐
│   S3 / GCS        │
│   (WAL archive +  │
│    base backups)  │
└───────────────────┘
```

That's the core. One primary, one standby, PgBouncer in front, WAL-archived to object storage.

## What Postgres Handles

In this architecture, Postgres handles everything by default:

**Core relational data:** Users, orders, products, accounts — your domain model, normalized, with proper foreign keys and constraints.

**Document storage:** Product attributes, configuration, event payloads — JSONB columns alongside relational columns. GIN indexes for queries.

**Full-text search:** Article content, product descriptions, support tickets — tsvector columns with GIN indexes. `websearch_to_tsquery` for user-facing search.

**Background jobs:** Order processing, email sending, report generation — a `jobs` table with `SKIP LOCKED`, or River/pg-boss. Transactional enqueue: jobs created atomically with the business data they operate on.

**Sessions and ephemeral data:** User sessions, rate limiting counters, temporary tokens — unlogged tables for performance with acceptable durability trade-offs.

**Caching:** Computed summaries, denormalized read models — materialized views or application-managed cache tables. Refreshed on schedule via pg_cron.

**Feature flags:** Configuration, A/B test assignments — a `feature_flags` table, read at application startup or cached with a short TTL.

**Vector search:** Embeddings for semantic search, RAG pipelines, recommendation systems — pgvector with HNSW indexes. Hybrid search combining vectors with full-text and relational filters.

**Time-series data:** Application metrics, user events, audit logs — partitioned tables or TimescaleDB, with BRIN indexes and continuous aggregates.

**Audit trails:** Every data change recorded with before/after values — trigger-based audit tables, partitioned by month.

**Scheduled tasks:** Partition maintenance, expired session cleanup, materialized view refresh — pg_cron, all visible and manageable from SQL.

## The Configuration Baseline

A Postgres configuration for a 32GB production server:

```ini
# Memory
shared_buffers = 8GB
effective_cache_size = 24GB
work_mem = 64MB
maintenance_work_mem = 2GB

# Checkpoints
checkpoint_completion_target = 0.9
max_wal_size = 4GB
checkpoint_timeout = 15min

# WAL
wal_level = replica
wal_compression = on
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'

# Connections
max_connections = 200  # PgBouncer handles fan-out

# Autovacuum
autovacuum_max_workers = 6
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 2000

# Logging
log_min_duration_statement = 1000
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

# Stats
shared_preload_libraries = 'pg_stat_statements, pg_cron'
pg_stat_statements.track = all
```

Tune from this baseline based on your workload profile and actual performance data.

## The Extension Set

The extensions to install on day one:

```sql
-- Observability (MUST HAVE)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Fuzzy search and LIKE acceleration
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Partition management
CREATE EXTENSION IF NOT EXISTS pg_partman;

-- Scheduled tasks
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Cryptography
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Vector search (when you need it)
CREATE EXTENSION IF NOT EXISTS vector;

-- Geospatial (when you need it)
CREATE EXTENSION IF NOT EXISTS postgis;

-- TimescaleDB (when time-series volume warrants it)
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

These extensions are mature, widely-deployed, and add significant capability with minimal operational complexity.

## The Schema Foundation

Every application table should have:

```sql
-- The template:
CREATE TABLE <table_name> (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    -- ... business columns ...
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Auto-update updated_at:
CREATE TRIGGER trigger_<table_name>_updated_at
    BEFORE UPDATE ON <table_name>
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- The trigger function (once per schema):
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

For multi-tenant applications, every table additionally has:
```sql
tenant_id BIGINT NOT NULL REFERENCES tenants(id),
```
with RLS policies enforcing tenant isolation and indexes on `tenant_id` to keep those policies fast.

## The Migration Workflow

Every schema change goes through:

1. **Write SQL migration** with `CONCURRENTLY` for indexes, `NOT VALID` + `VALIDATE` for constraints
2. **Test against production-sized data** on a clone
3. **Deploy in phases** if old and new code can't coexist (expand, then contract)
4. **`SET lock_timeout = '5s'`** before any DDL that takes `ACCESS EXCLUSIVE`
5. **Monitor** during migration: `pg_stat_activity`, `pg_locks`, slow query log

The expand/contract pattern for column renames and type changes. Never drop a column in the same deployment cycle as the one that stops using it.

## The Backup Setup

```
Daily:    Full backup via pgBackRest → S3
Hourly:   Differential backup via pgBackRest → S3
Continuous: WAL archiving via pgBackRest → S3

Retention:
  Full backups: 4 weeks
  Differential: 7 days
  WAL: 7 days (enables PITR within that window)

Testing:
  Monthly: Full restore test to a staging environment
  After config changes: Verify archiving still works
```

The goal: restore any database state from the past 7 days in under 2 hours (RTO), with data loss no greater than the last WAL segment (RPO measured in seconds).

## The Observability Stack

```
pg_stat_statements → query performance dashboard
postgres_exporter  → Prometheus → Grafana
pgBadger           → daily slow query report (from logs)
pg_stat_activity   → real-time connection monitoring
```

Alerts on:
- Cache hit ratio < 95%
- Replication lag > 60 seconds
- Dead tuple percentage > 20%
- `idle in transaction` connections > 5 for > 30 seconds
- Connections > 150 (80% of max_connections=200)
- Checkpoint_req > 20% of total checkpoints (WAL pressure)

## The Security Baseline

```sql
-- Application credentials (not superuser):
CREATE ROLE myapp WITH LOGIN PASSWORD '...';
GRANT readwrite TO myapp;

-- Separate migration credentials:
CREATE ROLE migrations WITH LOGIN PASSWORD '...';
GRANT readwrite TO migrations;
GRANT CREATE ON SCHEMA public TO migrations;

-- Read replica credentials:
CREATE ROLE myapp_reader WITH LOGIN PASSWORD '...';
GRANT readonly TO myapp_reader;

-- RLS on multi-tenant tables:
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::bigint);

-- Force SSL:
-- pg_hba.conf: hostssl all all 0.0.0.0/0 scram-sha-256
--              hostnossl all all 0.0.0.0/0 reject
```

## The High-Availability Topology

For production workloads:

```
Primary (read/write)
    ↓ streaming replication (async)
Standby (read-only, hot standby)
    ↓ HAProxy or DNS
Application (routes writes to primary, reads to standby or primary)
```

For critical workloads where data loss is unacceptable:
- Synchronous replication to one standby
- The other standbys async (for read scaling without commit latency impact)

For managed environments: use the cloud provider's multi-AZ HA. AWS RDS Multi-AZ, Cloud SQL HA, Aurora — these handle failover automatically.

## When to Add Something

The reference architecture above has no Redis, no Elasticsearch, no Kafka. When does that change?

**Add a caching layer (Redis or Memcached) when:**
- You've profiled and found that Postgres is the actual bottleneck for read-heavy workloads
- Your application has data that is fetched very frequently but changes rarely, and the cost of cache invalidation logic is less than the cost of the database load
- You need sub-millisecond latency at a scale that Postgres unlogged tables can't provide

**Add Elasticsearch when:**
- You have hundreds of millions of documents and search relevance is a core product feature that requires sophisticated tuning
- Your analytics team needs faceted navigation with complex aggregations at a scale that Postgres can't support

**Add Kafka when:**
- You need durable, ordered event streaming between multiple independent systems
- Your outbox table pattern is generating replication lag that's affecting your primary's performance
- You're integrating with external systems that speak Kafka

**Add a data warehouse (ClickHouse, BigQuery, Snowflake) when:**
- Your analytics queries are scanning billions of rows and impacting primary/replica performance
- Your data team needs complex ad-hoc analytical queries over your full dataset

Notice the pattern: these additions are made in response to evidence, not in anticipation of theoretical scale. The starting point is Postgres. The additions happen when you've hit the actual limit.

## A Final Word

The engineers who run sophisticated systems on a single Postgres instance are not lazy. They're not unaware of the alternatives. They've made a deliberate choice: to invest deeply in one tool and use it well, rather than spread thin knowledge across many.

Postgres rewards that investment. The more you know about it — MVCC, the query planner, VACUUM, WAL, extensions — the more valuable it becomes. The knowledge compounds. The expertise transfers to every team and system you work with. The operational habits you build (proper indexing, safe migrations, autovacuum monitoring, backup testing) make you a better engineer on any system.

The thesis of this book is simple: most teams reach for specialized databases before they need to, and pay a complexity tax they don't have to pay. Postgres, properly understood and configured, handles a remarkable range of workloads — relational, document, search, queuing, key-value, vector, time-series — with a level of reliability and operational simplicity that no polyglot stack can match.

Use it well. Use all of it. And when you eventually do hit its limits — and you might, if you build something large enough — you'll know exactly where the limits are, and you'll have the expertise to make the right call about what to add next.

Postgres is enough. Now go build something.

---

## Quick Reference: The Postgres-First Decision Tree

```
Is this about storing and querying data?
├── Yes
│   ├── Does it fit on one machine at current scale? → Postgres
│   ├── Is it time-series with high ingest? → TimescaleDB (it's still Postgres)
│   ├── Is it geospatial? → PostGIS (it's still Postgres)
│   ├── Is it vectors? → pgvector (it's still Postgres)
│   └── Is it truly global with write latency requirements? → Evaluate Spanner/CockroachDB
│
├── Do you need a job queue?
│   └── Is it < 100k jobs/sec? → River or pg-boss (Postgres)
│
├── Do you need full-text search?
│   └── Is it < 100M docs, not a search product? → Postgres FTS
│
├── Do you need caching?
│   └── Have you measured and proven Postgres is too slow? → Maybe Redis
│
└── Do you need pub/sub?
    ├── For worker notification? → LISTEN/NOTIFY (Postgres)
    └── For distributed event streaming? → Kafka
```

The answer to "do you need X?" is almost always "try Postgres first."
