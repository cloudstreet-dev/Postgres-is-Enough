# Postgres as a Job Queue

The job queue is the canonical Redis use case. Redis is fast, Redis has pub/sub, Redis has sorted sets for priority queues. Every tutorial adds Redis as the "obvious" choice for background job processing. And so teams end up with Postgres for their data and Redis for their job queue — two databases to operate, two failure modes to handle, and crucially, no way to enqueue a job inside the same transaction that creates the data the job needs.

That last point is what makes Postgres-backed job queues not just viable but *better* for most applications. When you enqueue a job inside a Postgres transaction, the job is only visible to workers if and when the transaction commits. If the transaction rolls back, the job disappears too. This atomicity is something Redis queues fundamentally cannot provide.

Brandur Leach and Blake Gentry built River — a Postgres-native job queue in Go — to prove this point definitively. River handles millions of jobs per day in production, with sub-millisecond job insertion latency, reliable at-least-once delivery, and zero additional infrastructure dependencies.

## The Primitives: SKIP LOCKED and Advisory Locks

The key innovation in modern Postgres-backed job queues is `SKIP LOCKED`, introduced in PostgreSQL 9.5.

The naive approach to a job queue in any database is:

```sql
-- Worker picks up a job
SELECT * FROM jobs WHERE status = 'pending' ORDER BY created_at LIMIT 1 FOR UPDATE;
-- (update status to 'processing')
```

The problem: `FOR UPDATE` locks the selected row. When multiple workers run this query simultaneously, they queue up waiting for each other's lock. This is a thundering herd — poor concurrency under load.

`SKIP LOCKED` tells Postgres to skip rows that are currently locked by another transaction, instead of waiting for them:

```sql
-- Worker picks up a job, skipping rows other workers are processing
SELECT * FROM jobs
WHERE status = 'pending'
  AND scheduled_at <= now()
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

Now multiple workers can run this query concurrently with no contention. Each worker gets a different pending job. If there are no unlocked pending jobs, the query returns zero rows immediately — no waiting.

`SKIP LOCKED` is the foundation of every serious Postgres job queue implementation.

### Advisory Locks

Advisory locks provide an alternative coordination mechanism. Instead of row-level locks, advisory locks are named locks identified by a 64-bit integer. They're lighter than row locks and can be used to coordinate across processes:

```sql
-- Try to acquire a per-job advisory lock
-- pg_try_advisory_xact_lock returns false if already held, true if acquired
SELECT pg_try_advisory_xact_lock(id) FROM jobs
WHERE status = 'pending'
  AND scheduled_at <= now()
ORDER BY created_at
LIMIT 1;
```

Some job queue implementations combine these: `SKIP LOCKED` for fast selection among many workers, advisory locks for longer-running jobs where holding a row lock is expensive.

## Building a Job Queue From Scratch

Understanding the fundamentals is useful even if you use a library. Here's a minimal job queue implementation:

```sql
CREATE TYPE job_status AS ENUM ('pending', 'processing', 'completed', 'failed', 'discarded');

CREATE TABLE jobs (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    queue TEXT NOT NULL DEFAULT 'default',
    kind TEXT NOT NULL,
    args JSONB NOT NULL DEFAULT '{}',
    status job_status NOT NULL DEFAULT 'pending',
    priority INTEGER NOT NULL DEFAULT 0,
    max_attempts INTEGER NOT NULL DEFAULT 3,
    attempt INTEGER NOT NULL DEFAULT 0,
    attempted_at TIMESTAMPTZ,
    scheduled_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    finalized_at TIMESTAMPTZ,
    errors JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Index for workers to efficiently find work
CREATE INDEX idx_jobs_pending ON jobs(queue, priority DESC, scheduled_at)
    WHERE status = 'pending';
```

Worker fetch query:

```sql
-- Fetch and lock a job in one atomic operation
WITH selected AS (
    SELECT id FROM jobs
    WHERE queue = $1
      AND status = 'pending'
      AND scheduled_at <= now()
    ORDER BY priority DESC, scheduled_at
    LIMIT $2  -- batch size
    FOR UPDATE SKIP LOCKED
)
UPDATE jobs
SET status = 'processing',
    attempt = attempt + 1,
    attempted_at = now()
FROM selected
WHERE jobs.id = selected.id
RETURNING jobs.*;
```

Complete the job:

```sql
UPDATE jobs
SET status = 'completed',
    finalized_at = now()
WHERE id = $1 AND status = 'processing';
```

Retry a failed job:

```sql
UPDATE jobs
SET status = CASE
    WHEN attempt >= max_attempts THEN 'discarded'::job_status
    ELSE 'pending'::job_status
END,
errors = coalesce(errors, '[]'::jsonb) || jsonb_build_object(
    'attempt', attempt,
    'error', $2::text,
    'at', now()
)::jsonb,
scheduled_at = now() + (power(2, attempt) * interval '1 second'),  -- exponential backoff
finalized_at = CASE WHEN attempt >= max_attempts THEN now() END
WHERE id = $1;
```

Transactional enqueue (the key advantage):

```sql
-- In the same transaction as your business logic:
BEGIN;
INSERT INTO orders (user_id, total_cents) VALUES ($1, $2) RETURNING id INTO order_id;
INSERT INTO jobs (kind, args) VALUES ('send_order_confirmation', jsonb_build_object('order_id', order_id));
COMMIT;
-- The email job is only enqueued if the order insert commits.
-- No order → no job. No partial state.
```

## River: The Gold Standard

River (riverqueue.com) is a Postgres-native job queue built by Brandur Leach and Blake Gentry. It's written in Go, though the concepts apply broadly.

What River does well:

**High performance.** River is designed for throughput. It handles tens of thousands of jobs per second on a single Postgres instance. It batches inserts, uses `SKIP LOCKED` efficiently, and minimizes roundtrips.

**Transactional enqueue.** River's client accepts a `pgx.Tx` or `database/sql.Tx` and enqueues jobs within the caller's transaction. This is the killer feature.

```go
// In Go, with River:
tx, err := db.Begin(ctx)
if err != nil { return err }
defer tx.Rollback(ctx)

// Insert your business data
order, err := queries.WithTx(tx).InsertOrder(ctx, params)
if err != nil { return err }

// Enqueue job in same transaction
_, err = riverClient.InsertTx(ctx, tx, &OrderConfirmationArgs{OrderID: order.ID}, nil)
if err != nil { return err }

return tx.Commit(ctx)
// Job is only enqueued if both the order insert AND commit succeed.
```

**Reliability guarantees.** River provides at-least-once delivery. Jobs are not acknowledged until the worker completes them, and failed jobs are retried with exponential backoff. A configurable number of max attempts sends jobs to the "discarded" state.

**Multi-queue and priority.** Jobs can be assigned to named queues (e.g., "critical", "bulk", "email") with separate worker pools. Priority within a queue is configurable per job.

**Job uniqueness.** River supports unique jobs — inserting a job with a key that's already pending is a no-op. Useful for debouncing or deduplication.

**Scheduled jobs.** Insert a job with a future `scheduled_at` to delay execution. Build cron-like recurring jobs using the scheduled_at field or River's periodic job functionality.

**Schema and migrations.** River ships its own migration system and manages its own schema (`river_job`, `river_leader`, etc.).

The River project demonstrates that a Postgres-backed job queue can be a first-class infrastructure primitive, not a stopgap.

## pg-boss: The Node.js Contender

For Node.js applications, pg-boss is the equivalent of River. It provides:

- A rich API for enqueuing, scheduling, and handling jobs
- Transactional job creation via `boss.sendDebounced()` and `boss.insert()`
- Job state management, retry, exponential backoff
- Scheduled and deferred jobs
- Job completion handlers and result storage

```javascript
const boss = new PgBoss(connectionString);
await boss.start();

// Enqueue within a transaction
await boss.send('order-confirmation', { orderId: 123 }, { priority: 2 });

// Worker
boss.work('order-confirmation', async ([job]) => {
    await sendConfirmationEmail(job.data.orderId);
});
```

pg-boss has been battle-tested in production at considerable scale. It's a mature, actively maintained library.

## The Outbox Pattern: Guaranteed Event Publishing

A related pattern — the *transactional outbox* — uses the same Postgres-based atomicity for reliable event publishing:

```sql
CREATE TABLE outbox_events (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    aggregate_type TEXT NOT NULL,
    aggregate_id BIGINT NOT NULL,
    event_type TEXT NOT NULL,
    payload JSONB NOT NULL,
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Within a transaction:
```sql
BEGIN;
-- Business operation
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- Record the event in the same transaction
INSERT INTO outbox_events (aggregate_type, aggregate_id, event_type, payload)
VALUES ('transfer', 1, 'MoneyTransferred', '{"from": 1, "to": 2, "amount": 100}');
COMMIT;
```

A separate process (or Debezium, listening to WAL changes) reads the outbox table and publishes events to Kafka or another message broker. Because the event is written in the same transaction as the business operation, you're guaranteed that either both happen or neither does.

This is how you bridge the gap between Postgres and a message queue like Kafka: the outbox table is your reliable bridge.

## Dead Letter Queue and Observability

Production job queues need dead letter queues — a place where permanently-failed jobs land for human review:

```sql
-- Jobs that have exhausted all retries
SELECT id, kind, args, attempt, errors, finalized_at
FROM jobs
WHERE status = 'discarded'
ORDER BY finalized_at DESC;

-- Queue depth by queue and priority
SELECT queue, count(*) FILTER (WHERE status = 'pending') AS pending,
             count(*) FILTER (WHERE status = 'processing') AS processing,
             count(*) FILTER (WHERE status = 'failed') AS failed,
             count(*) FILTER (WHERE status = 'discarded') AS discarded
FROM jobs
GROUP BY queue
ORDER BY queue;

-- Average wait time (time from enqueue to processing start)
SELECT queue, kind,
       avg(extract(epoch from (attempted_at - created_at))) AS avg_wait_seconds
FROM jobs
WHERE status IN ('processing', 'completed')
  AND attempted_at IS NOT NULL
  AND created_at > now() - interval '1 hour'
GROUP BY queue, kind;
```

## Maintenance: Cleaning Up Completed Jobs

Completed and discarded jobs accumulate. Set up a periodic cleanup:

```sql
-- Delete jobs completed more than 7 days ago
DELETE FROM jobs
WHERE status IN ('completed', 'discarded')
  AND finalized_at < now() - interval '7 days';
```

River handles this automatically via its maintenance system. If you're rolling your own, this cleanup should run regularly — either via a cron job, a pg_cron schedule (Chapter 11), or a maintenance worker.

## Postgres vs. Redis/RabbitMQ: The Real Comparison

**Where Postgres job queues win:**
- Transactional enqueue — insert a job and business data atomically
- Joins — query jobs alongside business data in one query
- No sync required — jobs are visible immediately, same transaction
- ACID semantics — at-least-once delivery with strong consistency
- No additional infrastructure — one less database to operate, monitor, and back up
- Rich query capabilities — complex job routing, priority queues, scheduled jobs
- Point-in-time recovery works for jobs too

**Where Redis/RabbitMQ win:**
- Raw throughput at very high rates (millions of messages per second)
- Pub/sub broadcast patterns (Postgres has LISTEN/NOTIFY but it's not a full pub/sub)
- Streaming workloads (Redis Streams, RabbitMQ exchanges)
- Jobs that need sub-millisecond enqueue latency at extreme scale
- Interoperability with other systems that don't know about Postgres

**The honest threshold:** If you're doing fewer than ~100,000 jobs per second per instance, a Postgres job queue is not your bottleneck. River has been benchmarked at tens of thousands of jobs/second — this covers the vast majority of workloads. The teams that genuinely need Redis for job queueing are running massive throughput requirements, not the typical "send a welcome email" background job.

The ACID transaction story is not just convenience — it prevents entire categories of bugs. With a Postgres job queue, it's impossible to insert a job for a record that doesn't exist (because the insert would roll back). It's impossible to miss a job because the process crashed between the database write and the enqueue. These are real bugs that Redis-based job queues require application-level workarounds to handle.

For most applications: Postgres is enough for job queuing, and the atomic transactional enqueue makes it *better*.
