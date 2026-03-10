# Indexes — The Full Picture

If schema design is the decision you'll regret for years, index design is the decision you'll regret for months. Bad indexes mean slow queries, angry users, and 3am pages. Good indexes are invisible — queries are fast, the planner makes sensible choices, and the only reason you look at the index configuration is to add a new one.

Postgres has a richer index ecosystem than any other major relational database. B-tree is just the beginning. Understanding when and how to use GIN, GiST, BRIN, hash, partial, and expression indexes is the difference between a database that struggles at scale and one that handles it gracefully.

## How Indexes Work (The Part That Matters)

Before diving into index types, it's worth being precise about what an index is and what it costs.

An index is a separate data structure that maintains a sorted (or hashed) reference to rows in a table. When Postgres evaluates a query, the planner can choose to use an index instead of reading the entire table (a sequential scan). An index scan finds the relevant rows in the index, then fetches them from the heap.

**Indexes are not free.** They cost:
- **Space:** An index typically uses 10–30% of the table's data size, sometimes more
- **Write overhead:** Every `INSERT`, `UPDATE`, and `DELETE` must update all applicable indexes on the table
- **Maintenance:** Indexes accumulate dead entries (dead tuples) that must be vacuumed; heavily updated indexes bloat
- **Planning time:** More indexes means more time spent by the planner evaluating which to use

A table with 20 indexes does not perform 20 times better than a table with 1 index. Adding indexes beyond what your actual query patterns require makes writes slower and doesn't help reads. Index ruthlessly but purposefully.

**The selectivity principle:** Indexes are most valuable when they are *selective* — when they narrow down the search space significantly. An index on `status` in an orders table where 95% of orders are `'delivered'` will be ignored by the planner for queries that filter on `status = 'delivered'` (because a sequential scan would be faster). The same index is invaluable for queries filtering on `status = 'processing'` (2% of rows). Selectivity is the key concept.

## B-tree Indexes

The default. When you write `CREATE INDEX`, you get a B-tree. When you add a `PRIMARY KEY` or `UNIQUE` constraint, Postgres creates a B-tree index automatically.

A B-tree (balanced tree) maintains sorted values, allowing:
- Equality lookups: `WHERE email = 'alice@example.com'`
- Range scans: `WHERE created_at BETWEEN '2024-01-01' AND '2024-02-01'`
- Ordering: `ORDER BY last_name` (if the index is on `last_name`, Postgres can avoid sorting)
- Prefix matching: `WHERE name LIKE 'foo%'` (but NOT `WHERE name LIKE '%foo'`)
- `IS NULL` and `IS NOT NULL` checks

B-trees support all comparison operators: `<`, `<=`, `=`, `>=`, `>`, `BETWEEN`, `IN`.

### Multi-column B-tree Indexes

A multi-column index (composite index) is a B-tree ordered by multiple columns:

```sql
CREATE INDEX idx_orders_user_id_status ON orders(user_id, status);
```

The critical rule: **a composite index is used from left to right.** This index supports:
- `WHERE user_id = $1` (uses just the first column)
- `WHERE user_id = $1 AND status = $2` (uses both columns)
- `WHERE user_id = $1 AND status = $2 ORDER BY user_id` (uses both, avoids sort)

But NOT:
- `WHERE status = $1` (the leading column is `user_id`, not `status`)

The column order in a composite index matters. Generally, put the most selective column first, and match the order to your most common query patterns.

### Index Scan vs. Bitmap Index Scan vs. Sequential Scan

Postgres has three ways to use an index:

**Index Scan:** Walks the B-tree to find matching entries, then fetches each heap row individually. Fast for small result sets, slow for large ones (many random I/O operations).

**Bitmap Index Scan:** Scans the index for all matching TIDs (tuple IDs), builds a bitmap of heap pages that need to be visited, then fetches those pages in order. More cache-friendly than a plain index scan for larger result sets. Can combine multiple indexes with AND/OR logic.

**Sequential Scan:** Reads the entire table in physical order. Wins when a large fraction of the table matches the query.

The planner chooses based on cost estimates. If your query returns more than roughly 5–15% of the table, the planner often prefers a sequential scan. This is correct behavior — don't try to force index use on highly selective queries.

### Covering Indexes (Index-Only Scans)

If an index contains all the columns a query needs, Postgres can satisfy the query from the index alone — no heap visit required. This is an *index-only scan* and is significantly faster.

```sql
-- This query:
SELECT email, created_at FROM users WHERE created_at > now() - interval '7 days';

-- Is satisfied entirely by this index (no heap access):
CREATE INDEX idx_users_created_covering ON users(created_at) INCLUDE (email);
```

The `INCLUDE` clause (PostgreSQL 11+) adds columns to the index leaf pages without sorting by them. This makes them available for index-only scans without affecting the index's sort order or usability as a regular index.

Covering indexes can dramatically speed up read-heavy queries. The trade-off: they're larger than plain indexes, since they store additional column data.

**Caveat:** Index-only scans still need to check the visibility map. If many pages are marked not all-visible (because of recent changes), Postgres falls back to heap access even with a covering index. Regular VACUUM helps by updating the visibility map.

## GIN Indexes: For Multi-Valued Data

GIN (Generalized Inverted Index) indexes are designed for data types that contain multiple values per row, where you want to find rows that contain a particular value. The canonical use cases are:

- **Arrays:** find all rows where the array contains a value
- **JSONB:** find all rows where the JSON document contains a key or value
- **Full-text search:** find all rows where the tsvector contains a lexeme
- **`pg_trgm`:** trigram-based fuzzy string matching (requires the extension)

GIN indexes maintain a mapping from "value" to "list of rows containing that value" — exactly like the inverted indexes used in search engines. This is why they're expensive to update (every insert or update that changes the indexed values must update multiple postings lists) but fast for containment queries.

```sql
-- GIN index for JSONB:
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);

-- GIN index for arrays:
CREATE INDEX idx_articles_tags ON articles USING GIN (tags);

-- GIN index for full-text search:
CREATE INDEX idx_articles_fts ON articles USING GIN (to_tsvector('english', title || ' ' || body));
```

GIN operators include:
- `@>`: contains (does the JSON document / array / tsvector contain this value?)
- `<@`: is contained by
- `&&`: overlap (do two arrays/tsvectors share any elements?)
- `@@`: text search match (tsvector `@@` tsquery)
- `?`: does this key exist in the JSONB?
- `?|`, `?&`: does this key exist for any/all of these keys?

**GIN vs. B-tree for JSONB:** B-tree indexes on JSONB work for equality comparisons on specific paths (`metadata->>'status' = 'active'`). GIN indexes support the containment and existence operators (`metadata @> '{"status": "active"}'`). For querying specific paths with `->>`operators, a B-tree index on the expression is often better. For containment queries, GIN is the right choice.

### `gin_pending_list_limit`

GIN indexes maintain a "pending list" of recently-added entries that haven't been fully integrated into the main index structure. Reads trigger a cleanup if the pending list is too large. You can control this with `gin_pending_list_limit`. `VACUUM` and `ANALYZE` also clean up the pending list.

## GiST Indexes: Generalized Search Trees

GiST (Generalized Search Tree) is a framework for building custom index types. It's used for:

- **Geometric types:** `POINT`, `LINE`, `BOX`, `POLYGON`, `CIRCLE` — spatial queries like "find all rows within a bounding box"
- **Range types:** `INT4RANGE`, `TSTZRANGE`, etc. — "find all ranges that overlap with this range"
- **`pg_trgm`:** trigram similarity (also supports GIN)
- **PostGIS:** geospatial data

Unlike GIN (which is an exact inverted index), GiST uses a lossy index with false positives — the index narrows down candidates, but a recheck against the actual data is required. This makes GiST slightly slower for final results but allows it to handle much more complex data types.

```sql
-- Range overlap query using GiST:
CREATE INDEX idx_reservations_period ON reservations USING GIST (period);

SELECT * FROM reservations
WHERE period && '[2024-06-01, 2024-06-15)'::TSTZRANGE;

-- PostGIS spatial index:
CREATE INDEX idx_stores_location ON stores USING GIST (location);

SELECT * FROM stores
WHERE ST_DWithin(location, ST_MakePoint(-73.935, 40.730)::geography, 5000);
```

**GiST vs. GIN for text similarity:** `pg_trgm` supports both GIN and GiST. GIN is generally faster for equality and similarity queries; GiST supports `LIKE` and `ILIKE` queries in addition to similarity operators. For fuzzy string matching, GIN is usually the better choice.

## BRIN Indexes: Block Range Indexes

BRIN (Block Range Index) is a small, fast index type designed for columns where the physical storage order correlates with the column values. The canonical use case: a `created_at` timestamp on an append-only table.

BRIN works by dividing the table into ranges of pages (blocks) and storing the minimum and maximum value of the indexed column within each range. To find rows matching a range query, Postgres identifies which block ranges might contain matching rows and only reads those.

```sql
CREATE INDEX idx_events_created_at ON events USING BRIN (created_at);
```

BRIN indexes are tiny — often 100–1000x smaller than a B-tree on the same column. The trade-off is precision: BRIN only excludes whole block ranges, so it still reads more data than a B-tree in most cases.

**When to use BRIN:**
- Very large, append-only tables (logs, events, time series) where sequential reads are acceptable
- When disk space is critical
- When the table is so large that a B-tree index is itself massive and slow to build

**When NOT to use BRIN:**
- Randomly ordered data (BRIN provides no benefit if the data isn't physically correlated with the column)
- High-selectivity queries on small tables (B-tree wins)
- Frequent updates (BRIN performs poorly when rows are updated non-sequentially)

For time-series data specifically, BRIN is often the right choice for timestamp columns. A B-tree index on `created_at` for a billion-row events table is gigantic; a BRIN index on the same column is a few megabytes.

## Hash Indexes

Hash indexes use a hash table to map column values to heap locations. They're fast for equality comparisons (`=`) and don't support range queries or ordering.

```sql
CREATE INDEX idx_users_session_token ON users USING HASH (session_token);
```

Before PostgreSQL 10, hash indexes were not WAL-logged and couldn't be replicated. Since PostgreSQL 10, they're fully supported. That said, B-tree indexes support equality comparisons too, and also support ranges and ordering. The performance advantage of hash over B-tree for equality-only lookups is typically small — hash indexes may be slightly faster for very long key values (since they hash them rather than comparing byte-for-byte), but for normal string and integer keys, the difference is negligible.

**Verdict:** Hash indexes are rarely the right choice. Use B-tree unless you have specific benchmark evidence that hash is faster for your workload.

## Partial Indexes

A partial index is an index that covers only a subset of rows — those satisfying a `WHERE` condition:

```sql
-- Only index active users:
CREATE INDEX idx_users_active_email ON users(email) WHERE deleted_at IS NULL;

-- Only index pending jobs:
CREATE INDEX idx_jobs_pending ON jobs(scheduled_at) WHERE status = 'pending';
```

Partial indexes are powerful for several reasons:

**Smaller and faster:** A partial index on 10% of a table is 10% the size of a full index, and queries that include the matching condition can use it with great selectivity.

**Conditional uniqueness:** Enforce uniqueness only for specific cases:
```sql
-- Only one active subscription per user:
CREATE UNIQUE INDEX idx_subscriptions_active_per_user
    ON subscriptions(user_id)
    WHERE status = 'active';
```

**Exclude rare values from an index:** If 99% of rows have `status = 'processed'` and 1% have `status = 'pending'`, an index on `status` is nearly useless for the common case. A partial index on the rare case is tiny and precise.

The query must include the partial index's `WHERE` condition (or imply it) for Postgres to use it. The planner is smart about this — it recognizes that `status = 'pending' AND scheduled_at < now()` is compatible with a partial index defined as `WHERE status = 'pending'`.

## Expression Indexes

Index an expression rather than a column directly:

```sql
-- Case-insensitive email lookup:
CREATE INDEX idx_users_email_lower ON users(lower(email));

-- Index on JSONB field extraction:
CREATE INDEX idx_products_category ON products((metadata->>'category'));

-- Index on computed value:
CREATE INDEX idx_orders_total ON orders((subtotal + tax_cents));
```

For the planner to use an expression index, the query must use the same expression:

```sql
-- Uses the index:
SELECT * FROM users WHERE lower(email) = lower($1);

-- Does NOT use the index (different expression):
SELECT * FROM users WHERE email ILIKE $1;
```

Expression indexes are especially useful for:
- Case-insensitive lookups (`lower()`)
- JSONB field extraction (`metadata->>'field'`)
- Date truncation (`date_trunc('day', created_at)`)
- Computed business keys

## Index Bloat and Maintenance

Indexes accumulate dead entries just like tables do. When rows are updated or deleted, the old index entries become "dead" — they still take up space but don't point to live tuples. VACUUM cleans them up.

Heavily written tables (lots of `UPDATE` and `DELETE`) tend to have bloated indexes. You can check index bloat with queries against `pg_stat_user_indexes` and `pgstattuple` (requires the extension):

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT pgstatindex('idx_orders_user_id');
-- Returns: avg_leaf_density, leaf_fragmentation, etc.
```

When an index becomes badly fragmented, you have two options:
- `REINDEX INDEX CONCURRENTLY idx_orders_user_id`: rebuilds the index without locking (PostgreSQL 12+)
- `DROP INDEX CONCURRENTLY` + `CREATE INDEX CONCURRENTLY`: same effect, slightly more control

`REINDEX CONCURRENTLY` (and `CREATE INDEX CONCURRENTLY`) build the index in the background, allowing reads and writes to continue. The initial build takes longer than a blocking REINDEX, but it doesn't cause downtime.

**Checking for unused indexes:**

```sql
SELECT schemaname, relname, indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_relation_size(indexrelid) DESC;
```

Indexes with `idx_scan = 0` have never been used since the last statistics reset. They're taking up space and slowing down writes for no benefit. Drop them (carefully, after confirming they're truly unused).

Reset statistics since last restart with:
```sql
SELECT pg_stat_reset();
```

## Index Design Patterns

**The Three-Column Guideline:** Most queries can be satisfied by an index covering 1–3 columns. If you find yourself creating 5- or 6-column indexes, your query is probably doing too much, or your schema design needs work.

**Equity before range:** In composite indexes, put equality columns before range columns:
```sql
-- Good: user_id is equality, created_at is range
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- Less good: range before equality rarely helps
CREATE INDEX idx_orders_date_user ON orders(created_at, user_id);
```

**One index per query, not one query per index:** Design indexes around your actual query patterns, not hypothetically. Add an index when a slow query requires it, not preemptively.

**The `EXPLAIN ANALYZE` feedback loop:** The only reliable way to know if an index helps is to:
1. Run `EXPLAIN (ANALYZE, BUFFERS)` on the query without the index
2. Create the index
3. Run `EXPLAIN (ANALYZE, BUFFERS)` again
4. Compare execution time and buffer reads

Don't guess. Measure.

## Putting It Together: An Index Audit

When auditing a production system's indexes, check:

```sql
-- Indexes with no scans (candidates for removal)
SELECT indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Tables with high seq scan rates (may need indexes)
SELECT relname, seq_scan, idx_scan,
       round(idx_scan::numeric / nullif(seq_scan + idx_scan, 0) * 100, 2) AS idx_pct
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_scan DESC;

-- Duplicate indexes
SELECT indrelid::regclass AS table_name,
       array_agg(indexrelid::regclass) AS duplicate_indexes
FROM pg_index
GROUP BY indrelid, indkey
HAVING count(*) > 1;
```

The last query finds indexes with identical column definitions — a common result of ORMs creating indexes and DBAs creating the same indexes manually.

Indexes are one of the highest-leverage tools in Postgres. The investment in understanding them pays dividends every time you look at a slow query and immediately know what index to create — or what index is being ignored, and why.
