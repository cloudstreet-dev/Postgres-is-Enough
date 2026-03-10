# Postgres as a Document Store

In 2014, Jacob Kaplan-Moss published a post arguing that since the release of PostgreSQL 9.4 and its JSONB type, Postgres had become a better document database than MongoDB. He wasn't trolling. He was making a precise technical argument: Postgres could store, index, and query semi-structured JSON documents with a richer feature set, stronger consistency guarantees, and better performance characteristics than MongoDB at the time.

A decade later, that argument has only gotten stronger. Postgres's JSONB type has gained operators, functions, and a path language (jsonpath) that rivals or exceeds what any document database offers. And it comes with everything else Postgres provides — ACID transactions, joins against relational tables, mature replication, row-level security, and decades of operational knowledge.

This chapter explains how to use Postgres as a document store for real workloads.

## JSONB vs. JSON

Postgres has two JSON types:

**`JSON`:** Stores the JSON as-is, as text. Validates that it's well-formed JSON, then stores it verbatim. This means it preserves formatting, duplicate keys, and key order. It also means every time you query it, Postgres has to re-parse the text. You cannot index JSON columns (other than a B-tree index on the whole serialized string, which is useless).

**`JSONB`:** Stores JSON in a decomposed binary format. Parsing happens once on write, not on every read. Key order is not preserved (keys are stored sorted). Duplicate keys are deduplicated (last one wins). JSONB supports all the rich indexing and operator support. This is what you want.

Always use `JSONB`. There is almost no reason to use `JSON` over `JSONB` unless you specifically need to preserve duplicate keys or exact formatting.

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sku TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    attributes JSONB NOT NULL DEFAULT '{}'
);
```

## The JSONB Operator Zoo

JSONB comes with a large set of operators. The important ones:

### Navigation Operators

```sql
-- -> extracts a JSON object field by key, returns JSON
SELECT attributes -> 'color' FROM products WHERE id = 1;
-- Returns: "red" (JSON, quoted)

-- ->> extracts a JSON object field by key, returns TEXT
SELECT attributes ->> 'color' FROM products WHERE id = 1;
-- Returns: red (text, unquoted)

-- -> with integer extracts from a JSON array
SELECT attributes -> 'sizes' -> 0 FROM products WHERE id = 1;
-- Returns: "S" (first element, as JSON)

-- #> extracts a path, returns JSON
SELECT attributes #> '{dimensions, width}' FROM products WHERE id = 1;

-- #>> extracts a path, returns TEXT
SELECT attributes #>> '{dimensions, width}' FROM products WHERE id = 1;
```

### Containment Operators

These are the operators that make JSONB indexes useful:

```sql
-- @> contains: does the left side contain the right side?
SELECT * FROM products WHERE attributes @> '{"color": "red"}';

-- <@ is contained by
SELECT * FROM products WHERE '{"color": "red"}' <@ attributes;
```

Containment works recursively for nested objects:
```sql
-- Finds products where dimensions.width is 10
SELECT * FROM products WHERE attributes @> '{"dimensions": {"width": 10}}';
```

### Existence Operators

```sql
-- ? does this key exist?
SELECT * FROM products WHERE attributes ? 'color';

-- ?| does any of these keys exist?
SELECT * FROM products WHERE attributes ?| ARRAY['color', 'size'];

-- ?& do all of these keys exist?
SELECT * FROM products WHERE attributes ?& ARRAY['color', 'size'];
```

### Modification Operators (PostgreSQL 9.5+)

```sql
-- || concatenate (merge) two JSONB objects
UPDATE products
SET attributes = attributes || '{"in_stock": true}'
WHERE id = 1;

-- - remove a key
UPDATE products
SET attributes = attributes - 'discontinued'
WHERE id = 1;

-- #- remove at a path
UPDATE products
SET attributes = attributes #- '{dimensions, depth}'
WHERE id = 1;
```

### jsonb_set and jsonb_insert

For targeted updates to nested values:

```sql
-- jsonb_set(target, path, new_value, create_missing)
UPDATE products
SET attributes = jsonb_set(attributes, '{price, usd}', '29.99'::jsonb, true)
WHERE id = 1;

-- jsonb_insert(target, path, new_value, insert_after)
-- Inserts into an array at a position
UPDATE products
SET attributes = jsonb_insert(attributes, '{tags, 0}', '"featured"'::jsonb)
WHERE id = 1;
```

## Indexing JSONB

Raw JSONB queries without indexes require Postgres to scan every row and parse the JSONB for every document — equivalent to a MongoDB collection scan. Indexing makes JSONB queries fast.

### GIN Index on the Whole Column

A GIN index on a JSONB column indexes every key-value path in every document:

```sql
CREATE INDEX idx_products_attributes ON products USING GIN (attributes);
```

This single index supports:
- `WHERE attributes @> '{"color": "red"}'`
- `WHERE attributes ? 'color'`
- `WHERE attributes ?| ARRAY['color', 'size']`
- `WHERE attributes ?& ARRAY['color', 'size']`

The trade-off: a GIN index on a JSONB column with deeply nested, high-cardinality values can be large. If you have documents with hundreds of keys or deeply nested objects, the index can be many times larger than the data itself.

### Expression Index on a Specific Path

If you always query on a specific field, an expression index on that field is smaller and faster:

```sql
-- Index on a specific path as JSONB
CREATE INDEX idx_products_color ON products ((attributes -> 'color'));

-- Index on a specific path as TEXT (for ->> queries)
CREATE INDEX idx_products_color_text ON products ((attributes ->> 'color'));
```

Use these when:
- You have a high-cardinality field you frequently query (`user_id`, `order_number`)
- The GIN index would be too large for your document structure
- You want `ORDER BY` on a JSONB field to use an index

### GIN with `jsonb_path_ops`

The default GIN operator class for JSONB is `jsonb_ops`, which supports all the operators above. An alternative is `jsonb_path_ops`:

```sql
CREATE INDEX idx_products_attrs_path ON products USING GIN (attributes jsonb_path_ops);
```

`jsonb_path_ops` creates a smaller index (it only indexes paths, not keys separately) and is faster for the `@>` containment operator. But it doesn't support the `?`, `?|`, `?&`, and `@?` operators. If you primarily use containment queries, `jsonb_path_ops` is more efficient.

## jsonpath: The Query Language

PostgreSQL 12 introduced the `jsonpath` language — a powerful path expression language for navigating and filtering JSONB documents. If you're familiar with XPath for XML, jsonpath is the JSON equivalent.

```sql
-- @? Does the path exist? (like the ? operator)
SELECT * FROM products WHERE attributes @? '$.tags[*] ? (@ == "featured")';

-- @@ Does the path expression return true?
SELECT * FROM products WHERE attributes @@ '$.price.usd > 20';

-- jsonb_path_query returns all matches
SELECT jsonb_path_query(attributes, '$.variants[*].size')
FROM products
WHERE id = 1;

-- jsonb_path_exists
SELECT * FROM products
WHERE jsonb_path_exists(attributes, '$.tags[*] ? (@ starts with "eco")');
```

jsonpath supports:
- Path navigation: `$.field`, `$.nested.field`, `$.array[0]`, `$.array[*]`
- Filters: `? (@ > 5)`, `? (@ like_regex "pattern")`
- Methods: `.size()`, `.type()`, `.floor()`, `.ceiling()`, `.double()`, `.keyvalue()`
- Arithmetic: `+`, `-`, `*`, `/`, `%`
- Boolean: `&&`, `||`, `!`

jsonpath filters can also be indexed via GIN with `jsonb_path_ops`:

```sql
-- This query:
SELECT * FROM products WHERE attributes @? '$.price.usd ? (@ < 30)';

-- Can use a GIN index with jsonb_path_ops:
CREATE INDEX idx_products_attrs_path ON products USING GIN (attributes jsonb_path_ops);
```

## Functions for Working With JSONB

Beyond operators, Postgres has a rich set of JSONB functions:

```sql
-- Convert a table row to JSON
SELECT row_to_json(p.*) FROM products p WHERE id = 1;

-- Build a JSON object from key-value pairs
SELECT json_build_object('id', id, 'name', name) FROM products;

-- Build a JSON array from rows
SELECT json_agg(json_build_object('id', id, 'name', name)) FROM products;

-- Expand JSONB to a table (json_each)
SELECT key, value FROM jsonb_each(
    '{"color": "red", "size": "M", "weight": 150}'::jsonb
);

-- Expand a JSONB array to rows (jsonb_array_elements)
SELECT value FROM jsonb_array_elements('[1, 2, 3, 4, 5]'::jsonb);

-- Get all keys of a JSON object
SELECT jsonb_object_keys('{"a": 1, "b": 2}'::jsonb);
-- Returns: a, b

-- Pretty-print
SELECT jsonb_pretty('{"a":1,"b":{"c":2}}'::jsonb);

-- Strip nulls
SELECT json_strip_nulls('{"a": 1, "b": null, "c": 3}'::jsonb);
-- Returns: {"a": 1, "c": 3}
```

## Real-World Document Store Patterns

### Product Catalog With Variable Attributes

Classic e-commerce problem: products have different attributes depending on category. Shirts have size and color. Laptops have RAM and screen size. Books have ISBN and author. Storing all possible attributes in a single table with nullable columns creates a maintenance nightmare.

JSONB handles this elegantly:

```sql
CREATE TABLE products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    category TEXT NOT NULL,
    name TEXT NOT NULL,
    price_cents INTEGER NOT NULL,
    attributes JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Shirts
INSERT INTO products (category, name, price_cents, attributes)
VALUES ('apparel', 'Classic Tee', 2999,
        '{"color": "blue", "sizes": ["S", "M", "L", "XL"], "material": "cotton"}');

-- Laptops
INSERT INTO products (category, name, price_cents, attributes)
VALUES ('electronics', 'ProBook 15', 129999,
        '{"ram_gb": 16, "storage_gb": 512, "cpu": "Intel i7", "display_inches": 15.6}');

-- Index for category-specific queries
CREATE INDEX idx_products_attributes ON products USING GIN (attributes);

-- Find all blue apparel
SELECT name, price_cents
FROM products
WHERE category = 'apparel'
  AND attributes @> '{"color": "blue"}';

-- Find all laptops with at least 16GB RAM
SELECT name, price_cents
FROM products
WHERE category = 'electronics'
  AND (attributes ->> 'ram_gb')::integer >= 16;
```

### Schema-on-Read Configuration

A common use case for document storage: store configuration or settings objects where the schema evolves:

```sql
CREATE TABLE app_settings (
    tenant_id BIGINT NOT NULL REFERENCES tenants(id),
    settings JSONB NOT NULL DEFAULT '{}',
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id)
);

-- Read a specific setting with a default
SELECT COALESCE(settings -> 'notification' ->> 'email_digest_frequency', 'daily')
FROM app_settings
WHERE tenant_id = $1;

-- Update just one setting key
UPDATE app_settings
SET settings = jsonb_set(settings, '{notification, email_digest_frequency}', '"weekly"'::jsonb),
    updated_at = now()
WHERE tenant_id = $1;
```

### Event Sourcing / Audit Logs

Events and audit records often have variable payload structures that are natural to store as JSONB:

```sql
CREATE TABLE audit_events (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    event_type TEXT NOT NULL,
    actor_id BIGINT NOT NULL,
    entity_type TEXT NOT NULL,
    entity_id BIGINT NOT NULL,
    payload JSONB NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_audit_events_actor ON audit_events(actor_id, occurred_at DESC);
CREATE INDEX idx_audit_events_entity ON audit_events(entity_type, entity_id, occurred_at DESC);
CREATE INDEX idx_audit_events_payload ON audit_events USING GIN (payload);

-- Find all events where a specific field changed
SELECT * FROM audit_events
WHERE entity_type = 'order'
  AND payload @> '{"changed_fields": ["status"]}';
```

## Mixing Relational and Document Models

The real power of JSONB in Postgres is that you don't have to choose between relational and document models — you can use both in the same database, with joins between them.

```sql
-- A user has a relational identity, but flexible profile data
SELECT
    u.id,
    u.email,
    u.profile ->> 'display_name' AS display_name,
    u.profile ->> 'bio' AS bio,
    count(o.id) AS order_count,
    sum(o.total_cents) AS lifetime_value
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.profile @> '{"newsletter_opted_in": true}'
GROUP BY u.id;
```

This query joins across relational and JSONB data in a single pass. No data synchronization, no ETL pipeline, no application-layer join. You cannot do this in MongoDB without a separate `orders` collection and an application-side join.

## Postgres vs. MongoDB: An Honest Comparison

Jacob Kaplan-Moss's claim deserves examination. Where does each tool actually win?

**Postgres wins:**
- ACID transactions across multiple documents
- Joining documents with relational data
- Rich indexing options (multiple index types, partial indexes, expression indexes)
- SQL — a query language everyone knows, with window functions, CTEs, aggregates
- Full-text search alongside documents (Chapter 6)
- RLS for document-level access control
- One less database to operate
- Mature ecosystem of monitoring, backup, and replication tools
- jsonpath is as expressive as MongoDB's query language for most operations

**MongoDB wins:**
- Horizontal write scaling with sharding (Postgres requires Citus or manual sharding)
- Schema-less by default — no DDL required to add fields
- Native document-centric programming model — insert a nested object exactly as your application sees it
- Aggregation pipeline syntax (some find it more intuitive than SQL for document transformations)
- Easier to start with if your team doesn't know SQL
- Change streams for reactive document subscriptions (logical replication can provide similar capabilities in Postgres, but it's more complex)

**The honest assessment:** For most applications that chose MongoDB to avoid relational schema design or to store semi-structured data, Postgres + JSONB is the better choice today. The flexibility advantages of MongoDB are largely available in Postgres, and Postgres's consistency and operational advantages are substantial.

The cases where MongoDB makes more sense: write-heavy workloads at extreme scale that would require sharding, teams that are genuinely more productive with a document-centric model, or systems where the aggregation pipeline is a better fit than SQL for complex document transformations.

For the vast majority of CRUD applications storing semi-structured data: Postgres is enough.

## Performance Considerations

### Document Size

JSONB documents larger than about 8KB will be TOASTed (stored in the overflow table). This is transparent but adds overhead on reads. Very large documents (hundreds of KB or larger) should prompt you to reconsider whether they should be stored in the JSONB column or in a separate table.

### Partial Updates

Postgres doesn't support in-place partial update of JSONB — every update rewrites the entire column value. For frequently-updated JSONB columns with large documents, this creates significant write amplification. Consider splitting frequently-updated fields into their own columns.

### Index Size

GIN indexes on JSONB can be large — easily larger than the data itself for documents with many unique values. Monitor index sizes and use expression indexes on specific paths when a full GIN index is overkill.

```sql
-- Check index sizes
SELECT
    relname AS table,
    indexrelname AS index,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE relname = 'products'
ORDER BY pg_relation_size(indexrelid) DESC;
```

The key insight is this: JSONB is a power tool. It's excellent for genuinely variable or schema-less data. It's a poor substitute for proper columns when you know the schema. Use it where it shines, and the result is a flexible, performant document store that lives right next to your relational data, with no synchronization required.
