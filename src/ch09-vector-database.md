# Postgres as a Vector Database

Every AI-powered feature generates embeddings. Your RAG pipeline encodes documents as vectors. Your semantic search bar converts queries to vectors. Your recommendation engine represents users and items as vectors. And the conventional wisdom says you need a specialized vector database — Pinecone, Weaviate, Milvus, Qdrant — to store and search them.

You don't. Not for most workloads.

`pgvector` is a PostgreSQL extension that adds a native vector type, vector operations, and multiple index types for approximate nearest neighbor (ANN) search. It integrates seamlessly with your existing Postgres tables — vectors live alongside the data they describe, joinable with no synchronization required, secured with row-level security, and backed up with everything else.

## pgvector Basics

Install and enable:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

Define a vector column with a fixed dimension:

```sql
CREATE TABLE documents (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    content TEXT NOT NULL,
    embedding vector(1536),  -- OpenAI text-embedding-3-small dimensions
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Insert a vector:

```sql
INSERT INTO documents (content, embedding)
VALUES ('PostgreSQL is a powerful open-source relational database', '[0.02, -0.15, 0.08, ...]');
```

In practice, you'll generate embeddings in your application code and pass them as arrays:

```python
# Python example with openai + psycopg2
from openai import OpenAI
import psycopg2

client = OpenAI()

def embed(text: str) -> list[float]:
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

# Insert
embedding = embed("PostgreSQL is a powerful database")
cur.execute(
    "INSERT INTO documents (content, embedding) VALUES (%s, %s)",
    (content, embedding)
)
```

## Vector Operators

pgvector supports three distance metrics:

```sql
-- L2 distance (Euclidean) — good for OpenAI embeddings
SELECT content, embedding <-> '[0.02, -0.15, 0.08, ...]'::vector AS distance
FROM documents
ORDER BY distance
LIMIT 10;

-- Inner product (negative cosine similarity when normalized) — faster on normalized vectors
SELECT content, embedding <#> '[0.02, -0.15, 0.08, ...]'::vector AS distance
FROM documents
ORDER BY distance
LIMIT 10;

-- Cosine distance — measures angle, invariant to magnitude
SELECT content, embedding <=> '[0.02, -0.15, 0.08, ...]'::vector AS distance
FROM documents
ORDER BY distance
LIMIT 10;
```

Which distance metric to use depends on how the embeddings were generated:
- **OpenAI embeddings:** Cosine similarity (or inner product on normalized vectors) is recommended by OpenAI
- **Sentence transformers:** Cosine similarity is standard
- **Custom embeddings:** Check your model documentation

You can also compute vector functions:

```sql
-- L2 norm (magnitude)
SELECT vector_norm(embedding) FROM documents WHERE id = 1;

-- Element-wise average of multiple vectors
SELECT avg(embedding) FROM documents WHERE category = 'database';
```

## Index Types: IVFFlat and HNSW

Without an index, pgvector does exact nearest neighbor search — scanning every row and computing the distance to the query vector. This is exact but O(n). For large tables, it's impractical.

pgvector provides two approximate nearest neighbor (ANN) index types:

### IVFFlat (Inverted File with Flat Compression)

IVFFlat divides the vector space into clusters (Voronoi cells) and, at query time, searches only the most relevant clusters. It trades some accuracy for much faster queries.

```sql
-- Create IVFFlat index
-- lists = number of cluster centroids; typically sqrt(n_rows) is a good starting point
CREATE INDEX idx_documents_embedding_ivfflat
    ON documents USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

**Tuning IVFFlat:**

- `lists`: Number of clusters. More lists = higher build cost but better accuracy at query time. Rule of thumb: `sqrt(n)` to `n/1000`.
- `probes` (per-query): How many clusters to search. Higher = more accurate but slower. Set per session: `SET ivfflat.probes = 10;`

**IVFFlat limitations:**
- Must be built after data is inserted (it learns cluster positions from existing data). Adding data after index creation may degrade accuracy unless the index is rebuilt.
- The number of `lists` should be set based on expected data volume.

### HNSW (Hierarchical Navigable Small World)

HNSW is a graph-based ANN index. It builds a multi-layer graph where each layer is a progressively coarser approximation of the data. Queries start at the top (coarse) layer and navigate toward the query vector, descending to finer layers.

```sql
-- Create HNSW index
CREATE INDEX idx_documents_embedding_hnsw
    ON documents USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

**HNSW parameters:**
- `m`: Number of connections per node in the graph. Higher = better recall, larger index, slower build. Default 16; 8–64 is typical.
- `ef_construction`: Search breadth during index construction. Higher = better recall, slower build. Default 64; 64–200 is typical.
- `ef_search` (per-query): Search breadth at query time. `SET hnsw.ef_search = 100;` Higher = better recall, slower queries. Default 40.

**HNSW vs. IVFFlat:**

| Aspect | IVFFlat | HNSW |
|--------|---------|------|
| Build time | Faster | Slower |
| Query time | Slower | Faster |
| Recall at low `lists`/`probes` | Lower | Higher |
| Incrementally updated? | Degrades over time | Handles inserts well |
| Memory usage | Lower | Higher |
| Best for | Large static datasets, memory-constrained | Dynamic datasets, high-throughput queries |

For most production workloads with continuous data ingestion, HNSW is the better choice. For batch-loaded, rarely-updated datasets, IVFFlat is more efficient.

## Measuring Recall

ANN search is approximate — it won't always return the true nearest neighbors. Recall measures what fraction of the true top-k neighbors are found:

```sql
-- Exact nearest neighbors (no index, full scan)
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, embedding <=> $1::vector AS distance
FROM documents
ORDER BY distance
LIMIT 10;

-- Approximate (with index)
SET enable_seqscan = off;  -- Force index use for comparison
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, embedding <=> $1::vector AS distance
FROM documents
ORDER BY distance
LIMIT 10;
```

Comparing results from both gives you a recall estimate. Acceptable recall depends on your use case — for recommendation systems, 90% recall is often fine; for factual RAG pipelines, you may want 99%+.

## Hybrid Search: Vectors + Full Text + Filters

The most powerful pgvector feature is seamless integration with the rest of Postgres. Real-world searches almost always combine semantic similarity with metadata filters.

```sql
-- Semantic search within a specific category, by a specific author
SELECT
    d.id,
    d.title,
    d.content,
    d.embedding <=> $1::vector AS semantic_distance
FROM documents d
WHERE d.category = $2
  AND d.published = true
  AND d.author_id = ANY($3::bigint[])
ORDER BY semantic_distance
LIMIT 20;
```

This runs the vector search and applies the filters in one query. In a dedicated vector database, you'd need to fetch more candidates (with approximate filtering support) or filter post-retrieval — less efficient and more complex.

### Reciprocal Rank Fusion (RRF) for Hybrid Search

Combining semantic search with full-text search using RRF gives better results than either alone:

```sql
WITH semantic AS (
    SELECT id, row_number() OVER (ORDER BY embedding <=> $1::vector) AS rank
    FROM documents
    WHERE published = true
    LIMIT 100
),
fulltext AS (
    SELECT id, row_number() OVER (ORDER BY ts_rank(search_vector, query) DESC) AS rank
    FROM documents,
         websearch_to_tsquery('english', $2) AS query
    WHERE search_vector @@ query
      AND published = true
    LIMIT 100
),
rrf AS (
    SELECT
        COALESCE(s.id, f.id) AS id,
        COALESCE(1.0 / (60 + s.rank), 0) +
        COALESCE(1.0 / (60 + f.rank), 0) AS score
    FROM semantic s
    FULL OUTER JOIN fulltext f ON f.id = s.id
)
SELECT d.id, d.title, rrf.score
FROM rrf
JOIN documents d ON d.id = rrf.id
ORDER BY rrf.score DESC
LIMIT 20;
```

Reciprocal Rank Fusion with a constant of 60 is a well-established algorithm for combining rankings. This query does full-text and semantic search simultaneously and merges the results — in a single SQL query, in a single database.

## RAG (Retrieval-Augmented Generation) with pgvector

pgvector is an excellent foundation for RAG pipelines:

```sql
-- Store document chunks with their embeddings
CREATE TABLE document_chunks (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    document_id BIGINT NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    chunk_index INTEGER NOT NULL,
    content TEXT NOT NULL,
    token_count INTEGER,
    embedding vector(1536),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (document_id, chunk_index)
);

CREATE INDEX idx_chunks_embedding ON document_chunks USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- Retrieve relevant chunks for a query
SELECT
    dc.content,
    d.title,
    d.url,
    dc.embedding <=> $1::vector AS distance
FROM document_chunks dc
JOIN documents d ON d.id = dc.document_id
WHERE d.accessible_to_user_id = $2  -- Row-level access control
ORDER BY distance
LIMIT 5;
```

This is a complete RAG retrieval layer in Postgres. Access control via RLS or join conditions ensures users only see documents they're allowed to see — something that requires additional application logic in dedicated vector databases.

## Multi-Vector and Sparse-Dense Hybrid Search

Some embedding approaches use multiple vectors per document (e.g., ColBERT, which stores one vector per token). pgvector supports this naturally:

```sql
CREATE TABLE colbert_embeddings (
    document_id BIGINT NOT NULL REFERENCES documents(id),
    token_index INTEGER NOT NULL,
    embedding vector(128),
    PRIMARY KEY (document_id, token_index)
);
```

Sparse-dense hybrid models (like SPLADE + dense embeddings) can also be implemented by storing a JSONB column for sparse vectors alongside the dense vector column.

## Dimensionality and Model Considerations

Vector dimensions range from 256 (compact models) to 3072 (OpenAI text-embedding-3-large). Larger dimensions generally give better semantic accuracy but have higher storage and compute costs.

Storage cost for a vector column:
- `vector(384)` — Sentence-BERT small: ~1.5KB per row
- `vector(1536)` — OpenAI text-embedding-3-small: ~6KB per row
- `vector(3072)` — OpenAI text-embedding-3-large: ~12KB per row

At 1 million documents with `vector(1536)`, the embedding column alone is ~6GB. At 100 million documents, it's 600GB. Plan your storage accordingly.

HNSW index size is typically 1–3x the size of the vector data it indexes.

## pgvector vs. Dedicated Vector Databases

**Postgres + pgvector wins when:**
- You need to filter by metadata alongside vector similarity (no sync required)
- You need transactional consistency — vectors and relational data change together
- You're storing < ~100M vectors (pgvector handles this range well)
- You need row-level security on vector search
- You want one database to operate
- You want to join vector results with relational data

**Dedicated vector databases (Pinecone, Weaviate, Qdrant, Milvus) win when:**
- You're operating at 100M+ vectors and need horizontal scaling
- You need sub-10ms ANN search at very high query rates (thousands of QPS)
- You need multi-tenancy with strong isolation at scale
- You need advanced reranking or neural retrieval features
- Your team is building a search product, not using search as one feature

**The practical threshold:** For most AI-powered features — RAG pipelines, semantic search, recommendation widgets — pgvector handles the scale. The applications that need Pinecone are the ones where vector search is the core product, not a supporting feature. If you're adding a "similar documents" feature to your blog platform or semantic search to your internal knowledge base, pgvector is enough.

The hybrid search story is where pgvector especially shines. Running semantic search, keyword search, and metadata filtering in a single query with ACID guarantees is genuinely difficult with dedicated vector databases. In Postgres, it's a CTE.

## Operational Notes

**Maintenance:** HNSW indexes don't require explicit maintenance. IVFFlat indexes should be rebuilt periodically if data distribution changes significantly.

**Parallel builds:** pgvector supports parallel index builds. Set `max_parallel_maintenance_workers` appropriately.

**Progress monitoring:**

```sql
SELECT phase, blocks_done, blocks_total,
       round(blocks_done::numeric / nullif(blocks_total, 0) * 100, 2) AS pct
FROM pg_stat_progress_create_index
WHERE relid = 'documents'::regclass;
```

**`SET ivfflat.probes` and `SET hnsw.ef_search`** can be set per session or per transaction for workload-specific tuning — fast approximate queries at low recall, high-recall queries for critical retrieval.

The emergence of pgvector represents the same arc as JSONB and SKIP LOCKED before it: a capability once requiring a specialized tool, now integrated into Postgres with a level of quality that meets production requirements for most workloads. The trend line is clear.
