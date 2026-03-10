# The Case for Enough

There is a moment every engineering team knows. You are sitting in a planning meeting, and someone — usually the person who just came back from a conference, or who just finished a blog post, or who has been waiting for the right opportunity — says *"we should use Redis for that."* Or Elasticsearch. Or Kafka. Or DynamoDB. Or MongoDB. The proposal is made with the best of intentions. The new tool does do something well. The demo is compelling.

And so you add it.

Six months later, you have five databases to monitor, five different failure modes to understand, five backup strategies to maintain, five sets of client libraries to keep updated, five mental models to context-switch between, and a team that is increasingly not sure what lives where.

This is the trap. And almost every team falls into it.

## The Database Proliferation Problem

The modern data infrastructure stack at a midsize startup often looks something like this:

- **PostgreSQL** (or MySQL) for the core relational data
- **Redis** for caching, sessions, rate limiting, and sometimes a job queue
- **Elasticsearch** (or OpenSearch) for full-text search
- **MongoDB** (or DynamoDB) for "flexible" document storage
- **ClickHouse** (or BigQuery, or Redshift) for analytics
- **Kafka** (or RabbitMQ) for message passing and event streaming
- **Pinecone** (or Weaviate) for vector search, now that everyone is building AI features

Each of these databases was added for a reason. Redis is legitimately fast. Elasticsearch legitimately has good search capabilities. MongoDB legitimately has a flexible schema. None of these choices were irrational in isolation.

But the aggregate is a disaster.

Consider what "adding a database" actually means for a team:

**Operational burden.** Each database needs to be provisioned, configured, monitored, backed up, and eventually upgraded. If you are running on Kubernetes, each one needs manifests, persistent volume claims, and health checks. If you are using managed services, each one is a line item on your cloud bill. These costs are not one-time — they compound indefinitely.

**Failure surface.** Each database is a potential outage. Each one has its own failure modes, its own recovery procedures, its own operational quirks that take years to fully understand. The probability of *something* being degraded at any given moment increases with each addition.

**Knowledge fragmentation.** Your team can only go deep on so many things. Every hour spent learning Elasticsearch internals is an hour not spent going deeper on Postgres. Expertise is finite. When you spread it across five databases, you end up with shallow knowledge of all of them and deep knowledge of none. This is when bugs become incidents, and incidents become disasters.

**Consistency hazards.** Data that spans multiple databases is data that is eventually inconsistent. You cannot do a two-phase commit across Postgres and Redis. You cannot get a transaction that atomically updates Elasticsearch and PostgreSQL. The moment you spread a logical entity across multiple stores, you have accepted eventual consistency — whether you meant to or not.

**Onboarding friction.** Every new engineer who joins your team has to learn not just your codebase but your entire data topology. "The products are in Postgres but the search index is in Elasticsearch and the product embeddings are in Pinecone and sessions are in Redis" is a sentence that should make you wince.

## The Specialization Illusion

The argument for specialized databases is seductive: *use the right tool for the job*. Redis is faster than Postgres for simple key lookups. Elasticsearch has richer text analysis than Postgres's full-text search. MongoDB's flexible schema accommodates unstructured data more naturally than Postgres's rigid tables.

These claims were more true in 2012 than they are today.

PostgreSQL has been on a relentless march of capability expansion. The JSONB type (introduced in PostgreSQL 9.4, released in 2014) made Postgres a genuinely competitive document store. The GIN index type makes JSONB queries fast. The jsonpath language gives you MongoDB-style query expressiveness. Jacob Kaplan-Moss summarized this cleanly: Postgres is a better Mongo than Mongo — and has been for years.

Full-text search with `tsvector`, `tsquery`, `ts_rank`, and proper language dictionaries handles the vast majority of search use cases. Not every app needs Elasticsearch. Many apps that use Elasticsearch would be better served by Postgres.

Extensions like `pgvector` have turned Postgres into a capable vector database. The job queue use case — once the canonical Redis argument — is handled beautifully by libraries like River and pg-boss, which use `SKIP LOCKED` and advisory locks to deliver reliable, transactional job queues. Unlogged tables can push key-value throughput into Redis territory for many workloads.

None of this means Postgres does everything better than every specialized tool. Chapter 19 is specifically about where Postgres is *not* enough, and it's written honestly. But the question is not "does Postgres beat Redis at every benchmark?" The question is "does Postgres do this well enough that avoiding the operational complexity of Redis is worth it?" Far more often than people realize, the answer is yes.

## What "Enough" Actually Means

"Enough" is not a consolation prize. It is a precision claim.

When we say Postgres is enough, we mean that for the overwhelming majority of use cases engineers reach for specialized databases to solve, Postgres can solve them well — with less operational overhead, with stronger consistency guarantees, with a single mental model, and with decades of battle-tested reliability.

"Enough" means:

- Fast enough (Postgres can handle tens of thousands of queries per second on commodity hardware with proper tuning)
- Flexible enough (JSONB, arrays, hstore, custom types)
- Searchable enough (full-text search, trigram matching, vector similarity)
- Queuing enough (SKIP LOCKED, advisory locks, transactional outboxes)
- Analyzable enough (window functions, CTEs, lateral joins, aggregations)

"Enough" also means: stop asking whether Postgres *can* do a thing, and start asking whether the alternative is worth the cost.

## The Hidden Costs of Polyglot Persistence

The term "polyglot persistence" — using multiple different database technologies within a single system — was coined as a design principle, not a warning. The idea was that different data has different characteristics and should be stored in stores optimized for those characteristics.

In practice, it is usually a liability.

Here is what the hidden costs look like in practice:

**Cross-cutting queries become ETL pipelines.** The moment your data lives in two places, any query that touches both requires either a data warehouse, a synchronization job, or an application-layer join. Simple questions like "show me all users who clicked this button and then purchased within 24 hours" require a pipeline when clicks are in Kafka and purchases are in Postgres.

**Transactions become distributed transactions.** If you want to debit an account in Postgres and send a notification via a Redis pub/sub in the same atomic operation, you cannot. You have to accept that one of these operations might fail while the other succeeds, and build compensating logic to handle that case. This is hard to get right and even harder to test.

**The development environment gets complicated.** A `docker-compose.yml` that spins up Postgres, Redis, Elasticsearch, and Kafka is a heavy thing. Local development requires more RAM. CI pipelines run slower. New engineers spend their first day getting the environment to start.

**Vendor lock-in multiplies.** Every managed database service you adopt is a commitment. Migrating off any one of them is painful. Migrating off five simultaneously is project-defining work.

## A Brief History of "Just Postgres"

The "just use Postgres" movement is not new, but it has been gaining momentum as Postgres's capabilities have expanded. A few landmark moments:

**2012–2014: The JSONB moment.** MongoDB had become enormously popular as the "flexible schema" alternative to relational databases. Then PostgreSQL shipped JSONB — a binary JSON storage format with full indexing support, operators, and functions. The argument for MongoDB suddenly required much more justification.

**2016–2018: The job queue case gets made.** Engineers publishing case studies showing Postgres-backed job queues handling millions of jobs per day, reliably, with full ACID guarantees. The `SKIP LOCKED` clause (PostgreSQL 9.5, 2016) was the key primitive that made this practical.

**2019–2021: The extension explosion.** PostGIS had long been the gold standard for geospatial data. TimescaleDB made Postgres a credible time-series store. pg_partman made partition management practical. The ecosystem started looking more like a platform than a database.

**2023–present: The vector moment.** pgvector brings approximate nearest neighbor vector search to Postgres with HNSW and IVFFlat indexes. Suddenly the argument for keeping AI feature data in Postgres alongside your application data becomes very practical.

Each of these moments is a case where the specialized-database argument got harder to make.

## Who This Book Is For

This book is for engineers who are tired of managing five databases. It is for architects who have been burned by polyglot persistence and want to make the case for simplicity. It is for teams that are starting fresh and want to resist the pull toward premature complexity.

It is not for engineers who have already made the right choice for genuinely specialized requirements. If you are building a globally distributed system with Spanner semantics, use Spanner. If you are doing real-time analytics on petabytes of streaming data, use ClickHouse. Chapter 19 will tell you exactly when to reach for something else, and mean it.

But if you are building a typical web application, an API, an internal tool, a data pipeline of reasonable scale — and you are wondering whether you *really* need that Redis cluster, that Elasticsearch instance, that MongoDB collection — this book will help you answer that question with confidence.

The answer, more often than you think, is: Postgres is enough.

## How to Read This Book

The chapters are designed to be read in order, but most can stand alone if you already have a working Postgres foundation.

Chapters 1–4 establish the foundation: the case for simplicity, how Postgres works internally, schema design principles, and indexes. These chapters matter even if you never read another page — the most common Postgres performance problems come from misunderstanding these fundamentals.

Chapters 5–11 make the case for Postgres as a replacement for specialized databases. Each chapter covers one use case in depth, explains the relevant Postgres features, and compares honestly against the specialized alternative.

Chapters 12–18 cover operations: migrations, tuning, replication, backup, security, observability, and operational ergonomics. These chapters are where the "one database" dividend pays off.

Chapter 19 is the honest chapter: where Postgres isn't enough, and what to reach for instead.

Chapter 20 ties everything together with a reference architecture.

Let's start with understanding what Postgres actually is — not just how to query it, but how it works.
