# Migrations Done Right

Schema migrations are the most dangerous routine operation in a production database. Done wrong, they cause downtime, lock tables, and corrupt data. Done right, they're invisible — the schema evolves while the application serves traffic, and nobody notices.

This chapter covers the tools, techniques, and mental models for migrating a live Postgres database without pain.

## The Migration Mindset

Every migration you write will run against a live database. Your application will be serving requests while Postgres executes your SQL. This changes how you think about schema changes.

The fundamental constraints:
1. **Locking:** DDL operations take locks. Some locks block reads and writes for the duration. A migration that takes `ACCESS EXCLUSIVE` on a busy table will queue behind all active transactions, then block everything that comes after it until it completes. If it takes 10 minutes to complete, everything waits 10 minutes.

2. **Application compatibility:** During a deployment, you may have old application code running against the new schema, or new application code running against the old schema — or both simultaneously (during a rolling deployment). Your migration must not break either.

3. **Irreversibility:** Dropping a column is immediate and permanent. Getting the data back requires restoring from backup. Don't drop things you might need.

4. **Performance:** Rebuilding an index on a 500-million-row table takes hours. Adding a NOT NULL constraint without a default rewrites the table. These operations have real time costs.

## Locking and Dangerous Operations

PostgreSQL takes different lock levels for different DDL operations. The ones that cause production problems:

### `ACCESS EXCLUSIVE` Lock (Blocks Everything)

```sql
-- These take ACCESS EXCLUSIVE:
ALTER TABLE orders DROP COLUMN legacy_field;  -- locks entire table
ALTER TABLE orders ALTER COLUMN amount TYPE BIGINT;  -- rewrites table
ALTER TABLE orders ADD CONSTRAINT check_amount CHECK (amount > 0) NOT VALID;  -- actually fine with NOT VALID
TRUNCATE TABLE orders;
DROP TABLE orders;
```

The danger: `ALTER TABLE` takes `ACCESS EXCLUSIVE` and waits for all existing transactions to finish first. On a busy production table, there may always be active transactions. When the `ALTER TABLE` sits waiting, it also blocks all new queries on the table. This creates a backlog that looks like an outage.

Fix: Set a `lock_timeout` before dangerous DDL:

```sql
SET lock_timeout = '5s';
ALTER TABLE orders ADD COLUMN shipped_at TIMESTAMPTZ;
-- If it can't acquire the lock within 5 seconds, it fails (doesn't wait and block)
```

Retry the migration during lower-traffic periods if it times out.

### Adding a Column

**PostgreSQL 11+:** Adding a nullable column with no default or a constant default is instant — Postgres doesn't rewrite the table.

```sql
-- Instant in PG11+:
ALTER TABLE orders ADD COLUMN notes TEXT;
ALTER TABLE orders ADD COLUMN processed BOOLEAN NOT NULL DEFAULT false;
```

**Pre-PostgreSQL 11:** Adding a column with a non-null default rewrites the entire table. This can take hours on large tables and locks the table the entire time.

Modern Postgres (11+) uses a metadata trick: it stores the default value in the system catalog and returns it for any row that doesn't have the column physically written. Only new rows get the column written. This is "invisible" to queries but avoids the rewrite.

### Adding a NOT NULL Constraint

Adding `NOT NULL` to an existing column in PostgreSQL requires a table scan to verify no NULL values exist. On large tables, this is slow and takes `ACCESS EXCLUSIVE`.

The safe pattern:

```sql
-- Step 1: Add with default, NOT VALID (no scan needed)
ALTER TABLE orders ADD COLUMN customer_email TEXT;

-- Step 2: Backfill existing rows
UPDATE orders SET customer_email = '' WHERE customer_email IS NULL;

-- Step 3: Add the constraint as NOT VALID (doesn't check existing rows)
ALTER TABLE orders ADD CONSTRAINT orders_customer_email_not_null
    CHECK (customer_email IS NOT NULL) NOT VALID;

-- Step 4: Validate the constraint (light scan, doesn't block queries in PG 12+)
ALTER TABLE orders VALIDATE CONSTRAINT orders_customer_email_not_null;
```

`NOT VALID` creates the constraint but skips the validation scan. `VALIDATE CONSTRAINT` then validates existing rows, but in PostgreSQL 12+, this only takes `SHARE UPDATE EXCLUSIVE` (doesn't block reads and writes).

### Creating Indexes

`CREATE INDEX` (without `CONCURRENTLY`) takes `SHARE` lock — it blocks writes but not reads. For large tables, this is still a problem.

`CREATE INDEX CONCURRENTLY` builds the index in the background, only taking brief locks at the start and end:

```sql
-- Always use CONCURRENTLY on production:
CREATE INDEX CONCURRENTLY idx_orders_customer_email ON orders(customer_email);
```

Caveats:
- `CREATE INDEX CONCURRENTLY` cannot run inside a transaction block (`BEGIN`/`COMMIT`)
- It takes longer than a blocking `CREATE INDEX`
- If it fails, it leaves behind an `INVALID` index that must be dropped manually:
  ```sql
  -- Find and drop invalid indexes:
  SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
  DROP INDEX CONCURRENTLY idx_orders_customer_email;  -- then retry
  ```

### Renaming a Column

Renaming a column is fast (it only updates the system catalog) but requires `ACCESS EXCLUSIVE`. More importantly, it breaks any application code that uses the old name.

Never rename a column while application code is deployed that uses the old name. Use the expand/contract pattern instead (see below).

## The Expand/Contract Pattern

The expand/contract pattern (also called the parallel change pattern) is the safe way to rename columns, change types, or do any transformation that requires old and new states to coexist.

**Example: Renaming `user_id` to `customer_id`**

**Phase 1: Expand** — add the new column, start writing to both:
```sql
ALTER TABLE orders ADD COLUMN customer_id BIGINT REFERENCES customers(id);
```

Update application code to write to both `user_id` and `customer_id`, and read from `customer_id` (falling back to `user_id` if null).

**Phase 2: Backfill** — populate the new column for existing rows:
```sql
UPDATE orders
SET customer_id = user_id
WHERE customer_id IS NULL;
```

On very large tables, do this in batches:
```sql
DO $$
DECLARE
    batch_size INT := 10000;
    rows_updated INT;
BEGIN
    LOOP
        UPDATE orders
        SET customer_id = user_id
        WHERE customer_id IS NULL
          AND ctid IN (
              SELECT ctid FROM orders WHERE customer_id IS NULL LIMIT batch_size
          );
        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;
        PERFORM pg_sleep(0.1);  -- brief pause to avoid overwhelming autovacuum
    END LOOP;
END$$;
```

**Phase 3: Cut over** — once all rows are populated and application code only uses `customer_id`, add the NOT NULL constraint and deploy the final application version that removes the old column reference:

```sql
ALTER TABLE orders ALTER COLUMN customer_id SET NOT NULL;
```

**Phase 4: Contract** — remove the old column:
```sql
ALTER TABLE orders DROP COLUMN user_id;
```

This can be done in a later migration, after confirming the new column is correct.

## Migration Tools

### golang-migrate

Simple, widely-used, language-agnostic. Migrations are plain SQL files with up/down versions.

File naming: `001_create_users.up.sql`, `001_create_users.down.sql`

```sql
-- 001_create_users.up.sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 001_create_users.down.sql
DROP TABLE users;
```

```bash
migrate -path ./migrations -database $DATABASE_URL up
migrate -path ./migrations -database $DATABASE_URL down 1
```

golang-migrate tracks migration state in a `schema_migrations` table. Simple, reliable, integrates easily into CI/CD pipelines.

### Flyway

JVM-based, enterprise-grade, widely used in Java ecosystems. Supports complex migration scenarios, callbacks, undo migrations, and dry runs. Excellent integration with Spring Boot.

```sql
-- V1__Create_users.sql
CREATE TABLE users (...);

-- V2__Add_email_index.sql
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```

Flyway is the right choice if you're in the Java ecosystem or need enterprise features.

### Liquibase

Similar to Flyway but also supports XML/YAML/JSON migration formats in addition to SQL. Stronger focus on database-agnostic migrations. Overkill if you're Postgres-only.

### Atlas

A newer tool focused on declarative schema management. Instead of writing incremental migrations, you describe the desired schema state, and Atlas generates the migration SQL automatically.

```hcl
# schema.hcl
table "users" {
  schema = schema.public
  column "id" {
    type = bigint
    identity { generated = ALWAYS }
  }
  column "email" {
    type = text
  }
  primary_key { columns = [column.id] }
  unique "email_unique" { columns = [column.email] }
}
```

```bash
atlas schema apply --url $DATABASE_URL --to file://schema.hcl
```

Atlas has excellent Postgres support and generates safe migration SQL. Its `--dry-run` flag shows what changes would be made without applying them.

### ORM Migration Systems

ORMs like SQLAlchemy, Diesel, Prisma, and GORM include migration systems. They work well for simple cases but often generate unsafe SQL for production (blocking index creation, non-concurrent operations). Review and adjust ORM-generated migrations for large tables.

## Zero-Downtime Deployment Checklist

For a migration to be safe to run against a live production database:

- [ ] All DDL operations use `CONCURRENTLY` where applicable (indexes)
- [ ] All potentially long-running operations use `NOT VALID` / `VALIDATE CONSTRAINT` pattern
- [ ] `lock_timeout` is set before operations that take `ACCESS EXCLUSIVE`
- [ ] New columns are nullable or have constant defaults (not computed values)
- [ ] No `RENAME COLUMN` on a column currently in use by running application code
- [ ] No `DROP COLUMN` unless application code has been deployed that no longer uses it
- [ ] Large backfills are batched, not a single massive `UPDATE`
- [ ] The migration has been tested against a production-like data volume

## Rolling Back

Postgres has no built-in rollback for DDL that was committed. If a migration runs successfully but you realize it was wrong:

- **Additive changes** (new columns, new tables, new indexes) can be rolled back by removing them
- **Destructive changes** (dropped columns, dropped tables) require restoring from backup
- **Type changes** may require recreating the column and backfilling

This is why down migrations are important: they document what "undo" looks like. But they're only useful if the data hasn't changed since the up migration ran. A down migration that adds back a dropped column doesn't restore the data that was in it.

The safest rollback strategy: never drop a column in the same migration as the one that stopped using it. Leave dropped columns in place for at least one deployment cycle. This gives you time to roll back the application code if something is wrong, before the data is gone.

## Migration in Practice: The Workflow

1. **Write the migration.** Consider what lock it takes, how long it will run on your production data volume, and whether the new schema is backward-compatible with currently-deployed application code.

2. **Test on a production-sized clone.** Many migrations that seem fast in development are slow against 100 million rows. Test with realistic data volumes.

3. **Deploy in two phases if needed.** If old and new code can't coexist:
   - Phase 1: Expand (add new column, no constraints yet)
   - Deploy new application code (reads new column, writes both)
   - Phase 2: Contract (add constraints, drop old column)

4. **Monitor lock wait times.** During the migration, watch `pg_stat_activity` for blocked queries:
   ```sql
   SELECT pid, wait_event_type, wait_event, query, now() - query_start AS duration
   FROM pg_stat_activity
   WHERE state = 'active' AND wait_event_type = 'Lock'
   ORDER BY duration DESC;
   ```

5. **Verify completion.** Check that `pg_stat_progress_create_index`, `pg_stat_progress_copy`, etc. show the expected progress and finish.

Safe migrations aren't complicated — they require discipline, a few patterns (expand/contract, NOT VALID, CONCURRENTLY), and respect for what's running in production while you work. The teams that get burned by migrations are almost always the teams that wrote the SQL in development and ran it directly on production without considering the implications at scale.
