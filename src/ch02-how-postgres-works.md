# How Postgres Actually Works

You don't need to know how a database engine works to use it. But you do need to know how Postgres works to use Postgres *well*. The decisions you make about schema design, indexing, query structure, and configuration all become more coherent when you understand what's happening underneath. The bugs that confuse you — bloated tables, unexpected lock contention, query plans that make no sense — suddenly have obvious explanations.

This chapter is not an academic treatment of database internals. It's a working engineer's guide to the parts of Postgres that show up in production.

## Process Architecture

PostgreSQL is a process-per-connection database. When a client connects, Postgres forks a new process (called a *backend*) to handle that connection. This is different from MySQL's thread-per-connection model or the event-loop models used in databases like MongoDB.

The key processes in a running Postgres instance:

**Postmaster.** The root process. It listens for incoming connections, forks backends to handle them, and manages the background worker processes. If the postmaster dies, the whole instance goes down.

**Backend processes.** One per client connection. Each backend has its own memory for sorting, hash joins, and other operations — bounded by `work_mem`. Each backend can access shared memory (the buffer pool, etc.) but does its own query execution independently.

**Checkpointer.** Periodically writes dirty pages from the shared buffer cache to disk, ensuring durability. Checkpoints are important for recovery time — the longer between checkpoints, the more WAL replay is required after a crash.

**Background Writer.** Writes dirty pages to disk more continuously than the checkpointer, reducing the spike of I/O during checkpoints.

**WAL Writer.** Flushes the write-ahead log to disk. Without this, transactions are not durable.

**Autovacuum launcher and workers.** Manages the vacuum process across all tables. More on this in the performance chapter — autovacuum is one of the most important and most misunderstood parts of Postgres.

**Archiver.** If WAL archiving is configured, copies completed WAL segments to an archive location. Essential for point-in-time recovery.

**Stats collector.** Gathers statistics about table and index usage, query execution counts, and more — surfaced via `pg_stat_*` views.

The process model has implications for how Postgres scales. Because each connection is a full process, connections are expensive — they consume memory and create OS overhead. This is why connection pooling (PgBouncer, pgpool) is essentially mandatory for high-connection-count applications. The `max_connections` setting isn't just a configuration knob — it's a statement about how many concurrent backend processes you want the OS to manage.

## Shared Memory and the Buffer Cache

All backends share a region of memory called the *shared buffer cache* — configured by `shared_buffers`. When Postgres needs to read a page, it first checks this cache. If the page is there (a cache hit), it serves it from memory. If not (a cache miss), it reads it from disk into the cache.

The buffer cache uses a clock-sweep eviction algorithm. Pages that are frequently accessed get a higher "usage count" and are kept around longer. Pages that haven't been accessed recently get evicted to make room for new pages.

`shared_buffers` should typically be set to about 25% of available RAM for a dedicated Postgres server. The reason it's not higher is that the OS page cache also helps — Linux (and other operating systems) will cache frequently-read files in the OS page cache, which Postgres also benefits from. The total effective caching is `shared_buffers` plus whatever the OS is caching on top.

`effective_cache_size` is a hint to the query planner about how much total caching (shared_buffers + OS cache) is available. Setting it too low causes the planner to prefer sequential scans over index scans when it shouldn't.

## Storage: Heap Files and Pages

Postgres stores table data in *heap files* — ordinary files on disk, named after the table's OID (Object Identifier). Each table lives in one or more files in the data directory, under `$PGDATA/base/<database_oid>/`.

These files are divided into *pages* (also called *blocks*), each exactly 8KB by default. A page contains:

- A 24-byte header with metadata (checksum, flags, LSN, etc.)
- An array of *item pointers* (4 bytes each) that point to tuples within the page
- Free space in the middle (where new tuples go)
- *Tuples* at the end of the page, growing toward the middle

A *tuple* is Postgres's internal name for a row. Each tuple includes:

- A *tuple header* (`HeapTupleHeaderData`): visibility information, OID if applicable, number of attributes, null bitmap
- The actual column data

The separation of item pointers from tuples allows Postgres to move tuples within a page for vacuuming without invalidating the item pointer offset that indexes use to locate the tuple.

## MVCC: The Heart of Concurrency

Multi-Version Concurrency Control is the mechanism by which Postgres allows readers and writers to run concurrently without blocking each other. Understanding MVCC explains a lot of behaviors that otherwise seem mysterious.

The core idea: instead of updating a row in place (which would require locking readers out), Postgres creates a *new version* of the row. The old version remains visible to transactions that started before the update. The new version is only visible to transactions that start after the update commits.

### Transaction IDs and Visibility

Every transaction gets a *transaction ID* (XID), a 32-bit integer. Postgres uses XIDs to track which transactions created and invalidated each tuple.

Each tuple has two system columns:

- `xmin`: the XID of the transaction that created this tuple
- `xmax`: the XID of the transaction that deleted (or updated) this tuple; 0 if the tuple is "live"

When you update a row, Postgres:
1. Marks the old tuple's `xmax` with the current transaction XID
2. Creates a new tuple with the updated data, stamping its `xmin` with the current transaction XID

When you delete a row, Postgres:
1. Marks the tuple's `xmax` with the current transaction XID
2. Does *not* immediately reclaim the space — the tuple remains on disk until VACUUM removes it

This is why Postgres tables can grow even when you're not adding rows. Heavy `UPDATE` and `DELETE` workloads create dead tuples that accumulate until VACUUM cleans them up.

### Snapshots

When a transaction starts, Postgres takes a *snapshot* of the current state of the transaction ID space. The snapshot records:

- The current maximum assigned XID (`xmax`)
- A list of XIDs that are currently in progress (active but not committed)

Using this snapshot, a transaction can determine whether any given tuple is visible to it:

- If `tuple.xmin` > `snapshot.xmax`: this tuple was created *after* the snapshot — not visible
- If `tuple.xmin` is in the active list: this tuple was created by a transaction that wasn't committed when the snapshot was taken — not visible (with exceptions for read-committed isolation)
- If `tuple.xmax` is committed and < `snapshot.xmax` and not in the active list: this tuple was deleted before the snapshot — not visible
- Otherwise: visible

This logic is evaluated for *every tuple* during a scan. It's not free, but it's fast, and it enables the most important Postgres property: **readers never block writers, and writers never block readers**.

### Isolation Levels

Postgres supports four isolation levels, though it implements them in interesting ways:

**Read Committed (default).** Each statement within a transaction sees a fresh snapshot — committed rows from all transactions that committed before the statement started. This means within a single transaction, two identical `SELECT` statements can return different results.

**Repeatable Read.** The snapshot is taken at the start of the first statement in the transaction and held for the duration. All statements in the transaction see the same consistent view of the data. In Postgres, this is implemented using the same snapshot mechanism as serializable — there's no separate "lock all reads" implementation.

**Serializable (Serializable Snapshot Isolation, SSI).** Full serializable isolation, implemented with predicate locks that detect serialization anomalies and abort transactions that would violate them. Postgres's SSI implementation is genuinely impressive — it provides true serializability without the throughput collapse of traditional two-phase locking.

**Read Uncommitted.** Postgres doesn't actually implement this — it falls back to Read Committed. PostgreSQL never shows uncommitted data from other transactions.

### The XID Wraparound Problem

Transaction IDs are 32-bit integers. That means Postgres can only have about 4 billion unique XIDs. When the counter wraps around, old XIDs would appear newer than they are, causing Postgres to make all your data invisible.

Postgres prevents this with a concept called *freezing*. VACUUM periodically updates old tuples to mark them as "frozen" — visible to all transactions regardless of XID. This essentially removes them from the XID visibility system. The `autovacuum_freeze_max_age` setting controls how old a transaction can be before Postgres forces a freeze vacuum on the table.

XID wraparound is a real operational hazard. If autovacuum is disabled or blocked, and you hit `2^31` transactions since the last freeze vacuum, Postgres will:
1. Start emitting warnings at `autovacuum_freeze_max_age` transactions before wraparound
2. Stop accepting new connections entirely (going into "recovery mode") at approximately 3 million transactions before wraparound

Watch `pg_stat_user_tables.n_dead_tup` and `age(relfrozenxid)` in `pg_class`. Keep autovacuum healthy.

## The Write-Ahead Log (WAL)

The WAL is Postgres's mechanism for ensuring durability. Every modification to database state — inserts, updates, deletes, commits — is first written to the WAL before it is applied to the actual data files. If Postgres crashes mid-operation, it can replay the WAL on startup to recover to a consistent state.

The WAL is a circular log of *WAL records*, each describing a change to database state. WAL records are written to *WAL segments* — 16MB files (by default) in `$PGDATA/pg_wal/`. As segments fill up, new ones are created. Old segments can be archived (for point-in-time recovery) or deleted once they're no longer needed for recovery or replication.

WAL is also the mechanism for streaming replication. Standby servers connect to the primary and stream WAL records, applying them in order to maintain a replicated copy of the primary's state. This is why WAL and replication are inseparable concepts in Postgres.

### WAL and Durability

The `fsync` and `synchronous_commit` settings control durability vs. performance trade-offs:

**`fsync = on` (default):** After each transaction commits, Postgres calls `fsync()` to ensure the WAL is durable on disk. Without this, a crash could lose committed transactions.

**`synchronous_commit = on` (default):** The transaction doesn't return to the client until the WAL record is flushed to disk (and, if using synchronous replication, to the standbys). Setting this to `off` allows commits to return before fsync, improving throughput at the cost of a risk of losing the last few seconds of committed transactions on crash.

**`synchronous_commit = local`:** The transaction waits for the local WAL flush but not for standbys. A reasonable middle ground for some workloads.

The `wal_level` setting controls how much information is written to WAL:
- `minimal`: just enough for crash recovery
- `replica`: enough for streaming replication (the default since PostgreSQL 10)
- `logical`: enough for logical decoding and logical replication (required for tools like Debezium and pglogical)

## The Query Planner

Postgres's query planner is one of its defining features — and one of the things that requires the most understanding when things go wrong.

When you execute a query, Postgres goes through several phases:

1. **Parse:** Convert the SQL text into an abstract syntax tree
2. **Analyze/Rewrite:** Apply rewrite rules (views are expanded here), resolve names, type-check
3. **Plan:** Generate candidate execution plans, estimate costs, choose the best one
4. **Execute:** Run the chosen plan

The planner's job is to find the cheapest execution plan for your query. "Cheapest" means a cost estimate based on a model of:
- How many pages of data need to be read from disk
- How many tuples need to be processed
- How expensive each operation (sort, hash join, nested loop) is

### Statistics

The planner makes cost estimates using *statistics* collected by `ANALYZE`. For each column in each table, Postgres stores:

- `n_distinct`: estimated number of distinct values
- `correlation`: how well the physical order of the column matches the logical order (relevant for index scans)
- `most_common_vals` and `most_common_freqs`: the most common values and how often they appear
- A histogram of value distribution

Statistics are stored in `pg_statistic` (raw data, for internal use) and `pg_stats` (a friendlier view for humans).

The `default_statistics_target` setting (default: 100) controls how much detail to collect. For columns with high cardinality or skewed distributions where the planner keeps making bad estimates, increase the per-column statistics target:

```sql
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;
```

### Join Strategies

When joining tables, the planner has three strategies available:

**Nested Loop Join.** For each row in the outer relation, scan the inner relation. Efficient when the outer relation is small and the inner relation has a good index.

**Hash Join.** Build a hash table from the smaller relation, then probe it for each row in the larger relation. Great for large equality joins where both sides are big. Requires `work_mem` to hold the hash table.

**Merge Join.** If both relations are sorted on the join key (or can be sorted), merge them together in one pass. Efficient for large sorted datasets.

The planner chooses based on cost estimates. When the planner makes a wrong choice, it's usually because the statistics are stale or inaccurate. `EXPLAIN (ANALYZE, BUFFERS)` tells you what the planner *estimated* vs. what actually happened.

### Examining Plans

The `EXPLAIN` command is your window into the planner. Without `ANALYZE`, it shows estimated costs. With `ANALYZE`, it actually executes the query and shows real numbers.

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.email, count(o.id)
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.created_at > now() - interval '30 days'
GROUP BY u.email;
```

Key things to look for in a plan:
- **Rows estimates vs. actual rows**: large discrepancies indicate bad statistics
- **Seq Scans on large tables**: often a sign of a missing index
- **Nested loops on large outer relations**: usually a red flag
- **Buffers: hit vs. read**: cache hit rate for this query
- **Sort operations that spill to disk**: `work_mem` may be too low

`EXPLAIN (ANALYZE, BUFFERS)` is the most important debugging tool in Postgres. Learn to read it fluently.

## TOAST

Postgres has a maximum row size of approximately 8KB (one page). For columns that can hold larger values — text, bytea, jsonb, arrays — Postgres uses *TOAST* (The Oversized-Attribute Storage Technique).

TOAST works by:
1. **Compressing** the value inline if compression makes it fit in a page
2. **Chunking and storing** the value in a separate TOAST table if it's still too large

The TOAST table has a name like `pg_toast_<table_oid>` and is managed automatically. When you query a TOASTed column, Postgres automatically decompresses/reassembles the value.

TOAST has performance implications:
- Large JSONB documents, text fields, or binary data will be TOASTed, adding decompression overhead
- Selecting a TOASTed column from a wide table requires going to the TOAST table, not just the heap
- Index-only scans cannot use TOASTed column values from the visibility map — they need to visit the heap

The `STORAGE` attribute on a column controls TOAST behavior: `PLAIN` (never TOAST), `EXTERNAL` (TOAST without compression), `EXTENDED` (the default — compress first, then TOAST), `MAIN` (try to keep in heap but compress if needed).

## Locking

Postgres has a rich locking system that operates at multiple levels. Understanding it prevents production surprises.

### Table-Level Locks

Postgres has eight table-level lock modes, from weakest to strongest:
- `ACCESS SHARE`: taken by `SELECT` — compatible with almost everything
- `ROW SHARE`: taken by `SELECT FOR UPDATE/SHARE`
- `ROW EXCLUSIVE`: taken by `INSERT`, `UPDATE`, `DELETE`
- `SHARE UPDATE EXCLUSIVE`: taken by `VACUUM`, `ANALYZE`, some `ALTER TABLE`
- `SHARE`: taken by `CREATE INDEX` (not concurrently)
- `SHARE ROW EXCLUSIVE`: taken by some `CREATE TRIGGER`, `ALTER TABLE`
- `EXCLUSIVE`: taken by `REFRESH MATERIALIZED VIEW CONCURRENTLY`
- `ACCESS EXCLUSIVE`: taken by most `ALTER TABLE`, `DROP TABLE`, `TRUNCATE`, `VACUUM FULL`

The critical one for operations is `ACCESS EXCLUSIVE` — it blocks everything, including `SELECT`. Any DDL operation that takes `ACCESS EXCLUSIVE` will queue behind all active queries on the table and will block all subsequent queries while it waits. This is why `ALTER TABLE ADD COLUMN` (even for nullable columns, in older Postgres versions) could take down production.

### Row-Level Locks

Row-level locks are taken implicitly by DML. `UPDATE` and `DELETE` take a row-level exclusive lock on the affected rows. `SELECT FOR UPDATE` takes an explicit row-level lock for use in pessimistic locking patterns.

Row-level locks conflict only with each other — concurrent `UPDATE` on different rows does not block.

### Advisory Locks

Postgres also supports *advisory locks* — locks you explicitly acquire and release in application code. They're not tied to table rows; they're just named locks with a 64-bit integer key.

```sql
-- Acquire an advisory lock (blocks if another session holds it)
SELECT pg_advisory_lock(12345);
-- ... do work ...
SELECT pg_advisory_unlock(12345);

-- Try-acquire (returns false immediately if lock is held)
SELECT pg_try_advisory_lock(12345);
```

Advisory locks are per-session and are automatically released when the connection closes. They're the building block for distributed locking patterns and are used extensively in job queue implementations.

### Deadlocks

When two transactions each hold a lock the other wants, Postgres detects the deadlock and aborts one of the transactions (the one that's cheaper to abort). You can't prevent deadlocks entirely, but you can minimize them by always acquiring locks in the same order.

## Putting It Together

What you've learned in this chapter shapes every decision in the rest of the book:

- **MVCC** means reads don't block writes and vice versa — but dead tuples accumulate, and VACUUM is not optional
- **The buffer cache** means random I/O to recently-accessed data is fast — but cold data costs
- **The query planner** makes smart choices when statistics are accurate — keep statistics fresh with regular `ANALYZE`
- **The WAL** makes Postgres durable and replicable — it's also the foundation for logical replication and change data capture
- **TOAST** means you can store large values, but there are performance trade-offs
- **Locking** is fine-grained and avoids contention in most cases — but DDL operations can cause surprises

The single most important habit you can develop is running `EXPLAIN (ANALYZE, BUFFERS)` on slow queries and understanding what it tells you. Everything else in this book will make more sense because of that habit.

Next: schema design. The choices you make before the first row is inserted will shape your system for years.
