# Full Text Search

Full text search is the capability that teams add Elasticsearch for more than almost anything else. And it's the capability that, more often than not, Postgres can handle just as well — sometimes better — at a fraction of the operational complexity.

Elasticsearch is a remarkable piece of engineering. It's also a remarkable operational burden: a JVM cluster to provision and tune, heap sizes to configure, shard counts to get right, index mappings to manage, a query DSL to learn, and a synchronization pipeline to maintain between it and your primary database. If you're doing 100 million documents with complex relevance tuning and faceted navigation, Elasticsearch earns its keep. If you're doing full-text search over your application's content — blog posts, products, tickets, documents — Postgres might be all you need.

## How Postgres Full Text Search Works

Postgres full-text search is built around two types: `tsvector` and `tsquery`.

**`tsvector`** is a sorted list of *lexemes* — normalized word forms. When you convert text to a tsvector, Postgres:
1. Tokenizes the text into words
2. Normalizes each word (lowercasing, removing punctuation)
3. Applies the configured text search dictionary (which handles stop words, stemming, synonyms)
4. Stores the result as a sorted list of (lexeme, position_list) pairs

```sql
SELECT to_tsvector('english', 'The quick brown fox jumps over the lazy dog');
-- Returns: 'brown':3 'dog':9 'fox':4 'jump':5 'lazi':8 'quick':2
-- "The" and "over" are stop words (removed)
-- "jumps" → "jump", "lazy" → "lazi" (stemmed)
-- Positions are stored for phrase search
```

**`tsquery`** is a search query that can be matched against a tsvector:

```sql
SELECT to_tsquery('english', 'quick & fox');
-- Returns: 'quick' & 'fox'

SELECT to_tsquery('english', 'quick | fast');
-- Returns: 'quick' | 'fast'

SELECT to_tsquery('english', '!dog');
-- Returns: !'dog'  (NOT)

SELECT to_tsquery('english', 'quick <-> fox');
-- Returns: 'quick' <-> 'fox'  (phrase: "quick" followed immediately by "fox")

SELECT to_tsquery('english', 'quick <2> fox');
-- Returns: 'quick' <2> 'fox'  (proximity: "quick" within 2 positions of "fox")
```

The `@@` operator tests whether a tsvector matches a tsquery:

```sql
SELECT to_tsvector('english', 'The quick brown fox') @@ to_tsquery('english', 'quick & fox');
-- Returns: true
```

## Text Search Configurations

A text search configuration tells Postgres how to tokenize and normalize text. Configurations are language-specific — `english` handles English stemming and stop words, `spanish` handles Spanish, etc.

```sql
-- List available configurations
SELECT cfgname FROM pg_ts_config;

-- Show what a configuration does to text
SELECT * FROM ts_debug('english', 'The quick brown foxes are jumping');
```

You can create custom configurations for specialized vocabularies:

```sql
-- Create a custom configuration based on English
CREATE TEXT SEARCH CONFIGURATION custom_english (COPY = english);

-- Add a custom dictionary for domain-specific terms
-- (e.g., "PostgreSQL" → "postgres" + "sql")
```

For most applications, `english` (or the appropriate language configuration) is sufficient.

## `plainto_tsquery` and `websearch_to_tsquery`

Users typing search queries don't write `to_tsquery` syntax. Two alternatives convert natural language input:

**`plainto_tsquery`:** Converts a string to a tsquery treating all words as AND terms:
```sql
SELECT plainto_tsquery('english', 'quick brown fox');
-- Returns: 'quick' & 'brown' & 'fox'
```

**`websearch_to_tsquery`** (PostgreSQL 11+): Parses a web-search-style query with `"quoted phrases"`, `-excluded terms`, and `OR` operators:
```sql
SELECT websearch_to_tsquery('english', '"quick fox" OR lazy dog -cat');
-- Returns: 'quick' <-> 'fox' | ( 'lazi' & 'dog' ) & !'cat'
```

`websearch_to_tsquery` is the best choice for user-facing search — it handles the query syntax users already expect.

## Storing and Indexing tsvectors

Computing `to_tsvector()` on every query is expensive. The standard approach is to store the tsvector in a generated column and index it:

```sql
CREATE TABLE articles (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    author TEXT NOT NULL,
    published_at TIMESTAMPTZ,
    -- Generated tsvector column (automatically maintained)
    search_vector TSVECTOR GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(author, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(body, '')), 'C')
    ) STORED
);

-- GIN index on the stored tsvector
CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);
```

Generated columns (PostgreSQL 12+) maintain the tsvector automatically — you don't need triggers. When `title` or `body` changes, Postgres recomputes `search_vector` automatically.

For older Postgres versions, use a trigger:

```sql
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.body, '')), 'C');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_articles_search
    BEFORE INSERT OR UPDATE ON articles
    FOR EACH ROW EXECUTE FUNCTION update_search_vector();
```

## Ranking Results

Postgres can rank search results by relevance using `ts_rank` and `ts_rank_cd`:

**`ts_rank(vector, query)`:** Computes a relevance score based on how often and in how many fields the query terms appear.

**`ts_rank_cd(vector, query)`:** Cover density ranking — considers how close together the query terms appear in the document.

```sql
SELECT
    id,
    title,
    ts_rank(search_vector, query) AS rank
FROM articles,
     websearch_to_tsquery('english', 'postgres performance') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

`setweight` is how you boost certain fields. The weights are 'A' (highest), 'B', 'C', 'D':

```sql
-- Title matches count more than body matches
setweight(to_tsvector('english', title), 'A') ||
setweight(to_tsvector('english', body), 'C')
```

The `normalization` parameter to `ts_rank` controls how document length affects ranking:
- `0` (default): rank is not adjusted for document length
- `1`: rank / (1 + log(number of unique words))
- `2`: rank / document length
- `4`: rank / geometric mean of word count
- `8`: rank / unique word count
- `16`: rank / 1 + log(unique word count)
- `32`: rank / rank + 1

For most applications, `ts_rank(vector, query, 1)` (length-normalized) produces better results than the default.

## Highlighting Matches

`ts_headline` generates highlighted snippets showing where the query terms appear in the original text:

```sql
SELECT
    id,
    title,
    ts_headline('english', body, query,
        'StartSel=<mark>, StopSel=</mark>, MaxWords=30, MinWords=15, MaxFragments=3'
    ) AS snippet
FROM articles,
     websearch_to_tsquery('english', 'postgres indexing') AS query
WHERE search_vector @@ query
ORDER BY ts_rank(search_vector, query) DESC
LIMIT 10;
```

`ts_headline` is deliberately run against the original text (not the tsvector), which is why it can show context and formatting. Configuration options:
- `StartSel` / `StopSel`: HTML tags to wrap matching terms
- `MaxWords` / `MinWords`: length of each fragment
- `MaxFragments`: how many non-contiguous fragments to return
- `ShortWord`: minimum word length to highlight
- `HighlightAll`: highlight the entire document if no match is found

`ts_headline` is computationally expensive — it's parsing the original text, not the indexed tsvector. Call it only for the documents you're displaying, not in the `WHERE` clause.

## Trigram Search: Fuzzy Matching and `LIKE`

Standard full-text search doesn't handle typos or partial matches well. The `pg_trgm` extension adds trigram-based similarity, which does.

A trigram is a 3-character sequence. The word "postgres" generates trigrams: `pos`, `ost`, `stg`, `tgr`, `gre`, `res`. Similarity between two strings is measured by how many trigrams they share.

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Similarity score (0.0 to 1.0)
SELECT similarity('postgres', 'Postgre');
-- Returns: ~0.5

-- Distance (1.0 - similarity)
SELECT 'postgres' <-> 'Postgre';

-- Find similar strings
SELECT name FROM products
WHERE name % 'potsgres'  -- % is the similarity operator
ORDER BY name <-> 'potsgres';
```

GIN (or GiST) indexes on trigrams make `LIKE` and `ILIKE` queries fast — something normal B-tree indexes can't do for non-prefix patterns:

```sql
CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);

-- This now uses the index:
SELECT * FROM products WHERE name ILIKE '%postrges%';  -- handles typo
SELECT * FROM products WHERE name LIKE '%laptop%';
```

Trigram indexes are the right tool for:
- User-facing "search as you type" autocomplete
- Fuzzy matching (tolerate typos)
- Substring matching (`LIKE '%foo%'`)
- Similarity-based "did you mean?" suggestions

## Multi-table Full Text Search

Real applications search across multiple tables. The options:

### Union Approach

```sql
SELECT id, 'article' AS type, title, ts_rank(search_vector, query) AS rank
FROM articles, websearch_to_tsquery('english', 'postgres') AS query
WHERE search_vector @@ query

UNION ALL

SELECT id, 'product' AS type, name AS title, ts_rank(search_vector, query) AS rank
FROM products, websearch_to_tsquery('english', 'postgres') AS query
WHERE search_vector @@ query

ORDER BY rank DESC
LIMIT 20;
```

### Search Index Table

For complex multi-entity search, a dedicated search index table often works best:

```sql
CREATE TABLE search_index (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    entity_type TEXT NOT NULL,
    entity_id BIGINT NOT NULL,
    title TEXT NOT NULL,
    search_vector TSVECTOR NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_search_index_vector ON search_index USING GIN (search_vector);
CREATE INDEX idx_search_index_entity ON search_index(entity_type, entity_id);

-- Maintain via triggers or application logic
-- Query:
SELECT entity_type, entity_id, title,
       ts_rank(search_vector, query) AS rank
FROM search_index,
     websearch_to_tsquery('english', 'postgres performance') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

This pattern centralizes search logic and makes cross-entity search fast. The trade-off is maintaining the index table when source data changes.

## Dictionaries and Customization

Postgres's text search is highly customizable through dictionaries:

**Thesaurus:** Map terms to canonical equivalents. "Postgres" → "postgresql", "PG" → "postgresql".

**Synonym dictionaries:** Replace words with a set of synonyms during indexing.

**Ispell dictionaries:** Full morphological analysis dictionaries for specific languages.

**Stop word lists:** Words to ignore (articles, prepositions).

```sql
-- Create a thesaurus entry
CREATE TEXT SEARCH DICTIONARY my_thesaurus (
    TEMPLATE = thesaurus,
    DictFile = my_thesaurus,
    Dictionary = english_stem
);

-- Create a custom configuration using it
ALTER TEXT SEARCH CONFIGURATION custom_english
    ALTER MAPPING FOR asciiword WITH my_thesaurus, english_stem;
```

For most applications, the built-in language configurations are sufficient. Customization pays off for domain-specific vocabulary — medical, legal, technical terms.

## Practical Search API

Putting it together into a real search endpoint:

```sql
-- Full search with pagination, ranking, and snippets
WITH query AS (
    SELECT websearch_to_tsquery('english', $1) AS q
)
SELECT
    a.id,
    a.title,
    a.published_at,
    ts_rank_cd(a.search_vector, q.q, 1) AS rank,
    ts_headline(
        'english',
        left(a.body, 1000),  -- headline from first 1000 chars
        q.q,
        'MaxWords=20, MinWords=10, MaxFragments=2, StartSel=<em>, StopSel=</em>'
    ) AS snippet
FROM articles a, query q
WHERE a.search_vector @@ q.q
  AND a.published_at IS NOT NULL
ORDER BY rank DESC, a.published_at DESC
LIMIT $2
OFFSET $3;
```

## Postgres vs. Elasticsearch: An Honest Assessment

**Postgres full-text search wins when:**
- Your dataset fits on a single machine (up to hundreds of millions of documents)
- You want consistent search with your transactional data (no sync lag)
- You need to filter on relational data alongside search (user permissions, category filters)
- Search is one feature among many, not the core product
- You want to minimize operational complexity
- You need ACID-consistent search (immediately visible after insert)

**Elasticsearch wins when:**
- Horizontal scaling of search is required (multi-terabyte, billions of documents)
- Complex relevance tuning is a core product requirement (BM25 tuning, learning-to-rank)
- You need real-time faceted navigation with aggregations across large datasets
- Your team is already deeply invested in the Elastic ecosystem
- You have specific language analysis requirements that exceed Postgres's configurations

**The honest truth:** Most applications that use Elasticsearch don't need it. The typical use case — searching products, articles, tickets, comments across a few million records — works perfectly in Postgres. The teams that truly need Elasticsearch are running search as a primary product feature at scale. For everyone else, `websearch_to_tsquery` + a GIN index is enough.

The synchronization problem alone should give you pause. The moment you add Elasticsearch, you have two sources of truth — your Postgres primary and your Elasticsearch index. They will diverge. You'll have sync jobs that sometimes fail, search results that don't reflect recent changes, and bugs that only appear in the search path. None of that complexity exists when search lives in the same database as your data.

That said: Postgres full-text search is good, not magical. If you need deep relevance tuning, advanced language analysis for non-Latin scripts, or complex aggregated facets over billions of documents, Elasticsearch is the right tool. Acknowledge that threshold honestly and stay on the right side of it.
