# Where Postgres Isn't Enough

This book has argued, chapter by chapter, that Postgres can do more than most engineers realize — and that the operational cost of reaching for specialized databases is higher than it looks. That argument is true, and it's important.

But the argument for Postgres has limits. There are workloads and requirements where Postgres genuinely cannot compete, and where adding a specialized tool is the right call. Knowing where that line is — and being honest about it — is what separates principled engineering from dogmatism.

This chapter is about those limits. It will not hedge or equivocate. If you're in one of these situations, the right answer is not Postgres.

## Global Geographic Distribution

**The situation:** Your users are on five continents and you need reads and writes to complete with minimal latency for all of them. Latency from Tokyo to a Postgres primary in Virginia is 150ms+ at the speed of light — before any query processing. That's 300ms round trip for a write, minimum.

**Why Postgres can't solve this:** Postgres's primary-standby model means writes must go to the primary. You can have read replicas in every region, but writes still go to one place. For geographically distributed write workloads, you need multi-primary or multi-region write capabilities.

**What to reach for:** Google Spanner (or CockroachDB for open-source) provide globally-distributed SQL with multi-region writes and serializable transactions. YugabyteDB is another option.

**The honest threshold:** Most applications have users globally but write patterns that tolerate the round-trip to a single primary. If your p99 write latency of 300ms for Japanese users writing to a US primary is acceptable (it often is), Postgres with regional read replicas is fine. This limit applies when you need *sub-50ms write latency globally*, which is a hard requirement.

## Truly Extreme Write Throughput

**The situation:** You're ingesting millions of events per second — IoT sensors, financial tick data, high-frequency telemetry. Sustained write throughput of 100,000+ rows per second per second for sustained periods.

**Why Postgres can't solve this:** Postgres is a disk-based, WAL-journaled database with MVCC overhead. Each write must go through WAL sync, tuple versioning, and index maintenance. On excellent hardware (NVMe RAID), you might sustain 50,000-100,000 simple INSERTs per second with `synchronous_commit = off`. This isn't slow — but it's a ceiling.

**What to reach for:** Apache Cassandra or ScyllaDB for high-ingest append workloads (no transactions, eventual consistency). ClickHouse for high-ingest analytical workloads (columnar, batch-optimized). InfluxDB (or TimescaleDB on modern hardware) for time-series ingestion. Apache Kafka as a durability layer before writes reach any database.

**The honest threshold:** TimescaleDB + aggressive batching (COPY instead of INSERT) can handle millions of data points per minute. This covers most IoT and telemetry workloads. The hard limit is when you need millions of transactional rows per second with ACID guarantees — that's genuinely beyond what Postgres delivers.

## Complex Graph Traversal

**The situation:** Your data is fundamentally a graph — social networks, recommendation engines, fraud detection networks, knowledge graphs. You need to traverse relationships of arbitrary depth efficiently. "Find all users within 4 hops of user X" or "find the shortest path between nodes A and B."

**Why Postgres can't solve this:** Postgres can represent graphs (a nodes table, an edges table). Recursive CTEs let you traverse them. But recursive CTEs in Postgres are not optimized for graph traversal — they do level-by-level BFS/DFS and have no specialized graph index structures. For shallow traversals (2-3 hops) on reasonably sized graphs, Postgres works. For deep traversals on large graphs, query time grows exponentially.

**What to reach for:** Neo4j (or ArangoDB, TigerGraph) if graph traversal is a core product capability. The Cypher query language is genuinely expressive for graph queries in a way SQL recursive CTEs are not.

**The honest threshold:** If you need "find friends of friends" (depth 2) on a million-user social network, Postgres with proper indexes works fine. If you need "find all nodes reachable within 6 hops" on a billion-edge knowledge graph with sub-second latency, you need a dedicated graph database. Most applications don't have the latter requirement.

## Real-Time Streaming and Event Processing

**The situation:** You need to consume a stream of events, perform real-time aggregations and joins across the stream, and emit derived events — all with sub-second latency at high throughput.

**Why Postgres can't solve this:** Postgres is a request-response database, not a stream processor. The `LISTEN/NOTIFY` mechanism provides basic pub/sub, but it's not durable (messages are lost if no listener is connected), has no message ordering guarantees, and doesn't support stream processing patterns (windowed aggregations, stream joins).

**What to reach for:** Apache Kafka for durable, high-throughput event streaming. Apache Flink or Kafka Streams for real-time stream processing. RiverDB for event sourcing patterns that do fit within Postgres's capabilities.

**The honest threshold:** If your use case is "notify workers that there's new data" or "process events asynchronously," Postgres's job queue (Chapter 7) and LISTEN/NOTIFY are sufficient. If you need "join two high-throughput streams in real time and emit aggregated results within 1 second," that's a stream processor, not a database.

## Search at Scale With Complex Relevance Requirements

**The situation:** Search is your core product feature. You have hundreds of millions of documents, complex BM25 relevance tuning requirements, learning-to-rank with ML models, faceted navigation across many dimensions, and a team dedicated to improving search quality.

**Why Postgres can't solve this:** Postgres full-text search (Chapter 6) handles most search use cases well, but it's not designed for search-as-a-product at scale. The built-in ranking functions are less sophisticated than Elasticsearch's BM25 implementation. Faceted aggregations across hundreds of millions of documents under load require Elasticsearch's aggregation framework. Learning-to-rank is not a first-class concept.

**What to reach for:** Elasticsearch (or OpenSearch). Typesense or Meilisearch for simpler requirements with better operational simplicity.

**The honest threshold:** This limit applies to teams where search quality is a core competitive differentiator and a team is dedicated to optimizing it. For "good enough" search on a modest content corpus (millions of documents, not hundreds of millions), Postgres full-text search with `websearch_to_tsquery` and GIN indexes is excellent.

## Vector Search at Hundreds of Millions of Vectors

**The situation:** You have 500 million+ embeddings and need sub-10ms ANN search at thousands of queries per second with high recall.

**Why Postgres can't solve this:** pgvector (Chapter 9) handles tens to low hundreds of millions of vectors well on capable hardware. At hundreds of millions of vectors, HNSW indexes become very large (potentially hundreds of GB), index build time is significant, and query throughput at sub-10ms latency becomes difficult to guarantee on a single server.

**What to reach for:** Pinecone, Weaviate, Qdrant, or Milvus for dedicated high-scale vector search. These systems are purpose-built for horizontal scaling of vector indexes.

**The honest threshold:** For most RAG pipelines and semantic search features (tens of millions of document chunks), pgvector is enough. The teams that need dedicated vector databases are building search products where vector retrieval is the primary workload, not a supporting feature.

## Multi-Tenancy at Extreme Scale With Strong Isolation

**The situation:** You're building a SaaS product with thousands of tenants, each with millions of rows, and tenants require strict resource isolation — one tenant's heavy query load should not affect another tenant's latency.

**Why Postgres can't solve this:** Row-level security in a shared table (Chapter 16) provides data isolation but not resource isolation. If tenant A runs a massive analytical query, it competes for the same buffer cache and query threads as tenant B's transactional queries. Schema-per-tenant (separate schemas in one Postgres instance) provides better isolation but still shares resources. Database-per-tenant provides full isolation but operational overhead scales with tenant count.

**What to reach for:** For full resource isolation with large numbers of tenants: Citus (Postgres extension) for horizontal sharding, or dedicated Postgres instances per tenant (feasible with managed services like Supabase or Neon's serverless branching).

**The honest threshold:** For SaaS products with hundreds of tenants, shared-table RLS or schema-per-tenant in a single Postgres instance typically works fine. Resource isolation becomes a real issue when you have thousands of tenants with highly variable and potentially adversarial query loads.

## Analytics at Petabyte Scale

**The situation:** Your data warehouse has terabytes to petabytes of data. You need to scan it for analytics, run arbitrary aggregations, and support dozens of analysts simultaneously.

**Why Postgres can't solve this:** Postgres is a row-oriented database — data is stored in pages with all columns of a row together. Analytical queries that scan one or two columns from a billion-row table must still read all the row data. Columnar databases store data column-by-column, making analytical scans 10-100x more efficient.

**What to reach for:** ClickHouse (open-source, self-hosted) or BigQuery/Redshift/Snowflake (managed) for petabyte-scale analytical workloads. These are columnar databases optimized specifically for analytics.

**The honest threshold:** For reporting and analytics on operational data up to a few hundred million rows, Postgres with read replicas, partitioning, and materialized views is excellent. The pivot to a columnar warehouse makes sense when your analytics team needs to scan the entire fact table regularly, your data is in the tens of billions of rows, and query times become unacceptably long.

## Recognizing the Pattern

Notice what these limits have in common: they're all about *scale* or *streaming*, not capability. Postgres can do graphs, search, analytics, vector search, and time series — but at a scale threshold that most teams don't reach in the first several years of operation.

The failure mode isn't "Postgres can't do X." The failure mode is "we added Y database because we read that Postgres couldn't do X at 100x our actual scale." The requirement doesn't exist yet; the complexity does.

Before adding a specialized database, make the case based on evidence:
1. What is the actual current scale of this workload?
2. Have you measured Postgres's performance at this scale?
3. Is the bottleneck actually Postgres, or is it something else (network, application, query design)?
4. What is the operational cost of the specialized database, and is the performance gain worth it?

If you've done this analysis and the evidence is clear, add the specialized database. The teams that add databases preemptively, based on theoretical scale concerns, accumulate operational debt without evidence that the complexity is warranted.

Postgres is enough — until it isn't. Know the difference, and act on evidence rather than assumptions.
