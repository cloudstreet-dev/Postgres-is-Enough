# Postgres as a Key-Value Store

Redis is fast. There's no disputing that. A single Redis instance can handle a million operations per second on simple key-value workloads — that's an order of magnitude faster than what most Postgres setups deliver on random single-row reads. If you need that throughput profile, Redis is the right tool.

But that's not most Redis usage. The actual Redis usage in most production applications looks like this: sessions, rate limiting counters, feature flags, user preferences, configuration values, temporary tokens, and caches. Most of these workloads have a few hundred or a few thousand operations per second, not millions. And for those workloads, Postgres — with the right configuration — is competitive, consistent, and eliminates a database dependency.

## The HSTORE Type

PostgreSQL has a native key-value type: `HSTORE`. It stores a flat map of string key-value pairs as a single column value.

```sql
CREATE EXTENSION IF NOT EXISTS hstore;

CREATE TABLE user_preferences (
    user_id BIGINT PRIMARY KEY REFERENCES users(id),
    prefs HSTORE NOT NULL DEFAULT ''::hstore
);

-- Insert
INSERT INTO user_preferences (user_id, prefs)
VALUES (1, 'theme => dark, language => en, timezone => UTC');

-- Update a single key
UPDATE user_preferences
SET prefs = prefs || 'notifications_enabled => false'::hstore
WHERE user_id = 1;

-- Read a key
SELECT prefs -> 'theme' FROM user_preferences WHERE user_id = 1;

-- Check key existence
SELECT * FROM user_preferences WHERE prefs ? 'notifications_enabled';

-- Delete a key
UPDATE user_preferences
SET prefs = delete(prefs, 'deprecated_key')
WHERE user_id = 1;
```

HSTORE supports GIN and GiST indexing for key existence and containment queries:

```sql
-- Find all users with dark theme
CREATE INDEX idx_user_prefs ON user_preferences USING GIN (prefs);
SELECT user_id FROM user_preferences WHERE prefs @> 'theme => dark';
```

**HSTORE vs. JSONB:** HSTORE is a flat string→string map. JSONB supports nested structures, mixed types (numbers, booleans, arrays, objects), and richer query operators. For simple flat key-value storage, HSTORE is slightly more efficient. For anything with structure or mixed types, JSONB is better. In practice, most teams use JSONB for everything and HSTORE is rarely worth reaching for.

## Unlogged Tables: Trading Durability for Speed

The single most impactful Postgres feature for key-value-like performance is *unlogged tables*.

Normal Postgres tables write all changes to the WAL before writing to the heap. This durability guarantee is what makes Postgres crash-safe — but WAL writes have a cost. For data that's inherently ephemeral (sessions, caches, rate limiting counters, temporary computation results), WAL durability is pure overhead.

Unlogged tables bypass WAL writes entirely:

```sql
CREATE UNLOGGED TABLE sessions (
    token TEXT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    data JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_expires_at ON sessions(expires_at);
```

The performance difference is significant — unlogged tables are typically 5-10x faster for write-heavy workloads. The trade-off:

**On crash recovery:** Unlogged tables are truncated on database startup after an unclean shutdown. Any data in them is gone. This is acceptable for sessions (users just have to log in again), caches (they get rebuilt from the source of truth), and rate limiting counters (a brief reset is usually tolerable).

**Replication:** Unlogged tables are not replicated via streaming replication. Standbys have empty versions of unlogged tables. If you fail over to a standby, the unlogged data is gone on the standby.

Unlogged tables are a deliberate, intentional durability trade-off. Use them when the data is truly ephemeral.

```sql
-- A rate limiter using an unlogged table
CREATE UNLOGGED TABLE rate_limits (
    key TEXT NOT NULL,
    window_start TIMESTAMPTZ NOT NULL,
    request_count INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (key, window_start)
);

-- Increment counter, upsert-style
INSERT INTO rate_limits (key, window_start, request_count)
VALUES ($1, date_trunc('minute', now()), 1)
ON CONFLICT (key, window_start)
DO UPDATE SET request_count = rate_limits.request_count + 1
RETURNING request_count;
-- If request_count > limit, reject the request
```

## A Simple Key-Value API in Postgres

For applications that want a true key-value store in Postgres, a purpose-built table works well:

```sql
CREATE UNLOGGED TABLE kv_store (
    namespace TEXT NOT NULL DEFAULT 'default',
    key TEXT NOT NULL,
    value JSONB NOT NULL,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (namespace, key)
);

CREATE INDEX idx_kv_expires ON kv_store(expires_at)
    WHERE expires_at IS NOT NULL;
```

Get:
```sql
SELECT value FROM kv_store
WHERE namespace = $1 AND key = $2
  AND (expires_at IS NULL OR expires_at > now());
```

Set:
```sql
INSERT INTO kv_store (namespace, key, value, expires_at)
VALUES ($1, $2, $3::jsonb, $4)
ON CONFLICT (namespace, key)
DO UPDATE SET value = EXCLUDED.value,
              expires_at = EXCLUDED.expires_at,
              updated_at = now();
```

Delete:
```sql
DELETE FROM kv_store WHERE namespace = $1 AND key = $2;
```

Expire cleanup (run periodically via pg_cron or a background worker):
```sql
DELETE FROM kv_store WHERE expires_at < now();
```

This is a fully functional key-value store with TTL support, namespacing, and JSON values. Connection pooling (PgBouncer in transaction mode) makes the per-operation overhead minimal.

## Connection Pooling Is Mandatory

Postgres's process-per-connection model means connections are expensive. Each connection uses ~5-10MB of memory and involves OS process management overhead. For a key-value workload with many short operations, the bottleneck quickly becomes connection management, not query execution.

PgBouncer, running in *transaction mode*, solves this:

```
Application → PgBouncer → Postgres
(1000 app connections) → (20-50 server connections)
```

In transaction mode, PgBouncer assigns a Postgres connection to an application for the duration of a transaction, then returns it to the pool. For key-value operations (which are typically single-statement transactions), this means the connection is held for milliseconds, enabling many more effective operations per second.

Configure PgBouncer for a key-value workload:

```ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 10000   ; High — many app connections are fine
default_pool_size = 50     ; Low — fewer actual Postgres connections
min_pool_size = 10
reserve_pool_size = 5
server_idle_timeout = 600
```

With PgBouncer in transaction mode, a Postgres instance with 50 server connections can handle thousands of concurrent application connections with sub-millisecond key-value operations.

## Benchmark Reality

The throughput gap between Redis and Postgres narrows significantly with the right configuration:

- **Redis:** ~500,000–1,000,000 simple GET/SET ops/second on a single server
- **Postgres (standard table, no pooling):** ~5,000–15,000 ops/second on a single connection
- **Postgres (unlogged table + PgBouncer transaction mode):** ~50,000–200,000+ ops/second

The Postgres numbers vary widely with hardware, connection count, and query complexity. On modern NVMe SSDs with proper tuning, the gap is smaller than the benchmarks of five years ago suggest.

For most application use cases — sessions, feature flags, rate limiting, user preferences — the difference between 50,000 and 500,000 ops/second is irrelevant. Your application will never come close to saturating either system.

## Feature Flags in Postgres

Feature flags are a classic Redis use case. In practice, feature flag reads are infrequent relative to other database operations, and the latency is dominated by network round-trips rather than database processing time.

```sql
CREATE TABLE feature_flags (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    enabled BOOLEAN NOT NULL DEFAULT false,
    rollout_percentage SMALLINT CHECK (rollout_percentage BETWEEN 0 AND 100),
    allowed_user_ids BIGINT[],
    allowed_groups TEXT[],
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Check if a feature is enabled for a user
SELECT
    enabled AND (
        rollout_percentage = 100 OR
        (rollout_percentage IS NOT NULL AND
         hashtext(name || user_id::text) % 100 < rollout_percentage) OR
        (allowed_user_ids IS NOT NULL AND user_id = ANY(allowed_user_ids))
    ) AS is_enabled
FROM feature_flags
WHERE name = $1;
```

Most applications read feature flags at application startup or cache them at the application layer with a short TTL — avoiding per-request database reads entirely. The point is that Postgres stores the authoritative state, and your application layer caches it. This is simpler and more reliable than Redis.

## Sessions

Storing sessions in Postgres (rather than Redis or in-process memory) provides:
- Session persistence across application restarts
- Consistent session state across multiple application instances
- The ability to query and manage sessions (force-logout a user, list active sessions)

```sql
CREATE UNLOGGED TABLE http_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
    data JSONB NOT NULL DEFAULT '{}',
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_accessed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ NOT NULL DEFAULT now() + interval '30 days'
);

CREATE INDEX idx_sessions_user_id ON http_sessions(user_id)
    WHERE expires_at > now();
CREATE INDEX idx_sessions_expires ON http_sessions(expires_at);

-- Update last accessed (touch the session)
UPDATE http_sessions
SET last_accessed_at = now(),
    expires_at = now() + interval '30 days'
WHERE id = $1 AND expires_at > now()
RETURNING user_id, data;
```

Most session libraries for Node.js, Python, Go, and Ruby have Postgres adapters that implement this pattern. Using an unlogged table gives you Redis-like performance with no additional infrastructure.

## Caching Patterns in Postgres

Postgres itself is not a general-purpose cache (that's what `shared_buffers` and the OS page cache are). But *application-level caching tables* — storing the result of expensive computations or aggregations — can be implemented in Postgres:

```sql
CREATE UNLOGGED TABLE computed_cache (
    cache_key TEXT PRIMARY KEY,
    value JSONB NOT NULL,
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ NOT NULL
);

-- Cache aside pattern
-- 1. Check cache
SELECT value FROM computed_cache
WHERE cache_key = $1 AND expires_at > now();

-- 2. If miss, compute and store
INSERT INTO computed_cache (cache_key, value, expires_at)
VALUES ($1, $2::jsonb, now() + interval '5 minutes')
ON CONFLICT (cache_key) DO UPDATE
SET value = EXCLUDED.value,
    computed_at = now(),
    expires_at = EXCLUDED.expires_at;
```

This pattern is useful for expensive aggregations, denormalized read models, or computed statistics that change slowly. It's not a replacement for a full caching layer, but it eliminates a Redis dependency for many workloads.

## LISTEN/NOTIFY: The Lightweight Pub/Sub

Redis pub/sub is often used for simple real-time notification patterns — "notify workers that new data is available," "send a signal when a job finishes." Postgres has a built-in equivalent: `LISTEN` / `NOTIFY`.

```sql
-- Notify a channel
NOTIFY job_available, 'queue:default';

-- Or with PG's NOTIFY function in a trigger
CREATE OR REPLACE FUNCTION notify_job_available()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('job_available', NEW.queue);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_notify_jobs
    AFTER INSERT ON jobs
    FOR EACH ROW EXECUTE FUNCTION notify_job_available();
```

In application code, a worker can `LISTEN` on the channel and wake up immediately when a job is inserted, rather than polling:

```go
// Go example with pgx
conn.Exec(ctx, "LISTEN job_available")

for {
    notification, err := conn.WaitForNotification(ctx)
    // Process notification, fetch a job
}
```

`LISTEN/NOTIFY` is not a full pub/sub system — notifications are not durable, can be lost during reconnection, and don't support message history. But for "wake up a worker when there's work" patterns, it eliminates Redis entirely.

## Postgres vs. Redis for KV: The Honest Assessment

**Postgres wins when:**
- You need atomic operations with other relational data
- You need querying (find all sessions for a user, flag expiry, TTL management)
- Operational simplicity matters (one database to manage)
- Your throughput requirement is under ~100k ops/second
- You need persistence across database restarts (use a regular table, not unlogged)

**Redis wins when:**
- You need raw single-digit millisecond latency at scale
- You need millions of operations per second
- You need Redis-specific data structures (sorted sets, hyperloglogs, streams, bitmaps)
- You need pub/sub broadcast to many consumers
- You need LRU eviction policies that automatically manage memory

The key insight: most applications use Redis because it's easy to add and fast to demo, not because they've hit the throughput wall of a properly-configured Postgres. Unlogged tables + PgBouncer transaction mode cover the vast majority of KV workloads. The applications that genuinely need Redis's throughput profile are the exception, not the rule.

Start with Postgres. Add Redis when you can demonstrate the bottleneck.
