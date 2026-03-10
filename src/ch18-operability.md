# Operability Wins

The case for Postgres isn't only about what it can do. It's about what you don't have to do.

Operational simplicity is underrated in software engineering. The glamour is in new capabilities — "it can do vectors!" "it can queue jobs!" "it can search!" The cost is in maintenance — the 3am page when the Redis cluster gets into a split-brain, the week spent diagnosing why Elasticsearch's JVM heap is growing, the Monday spent carefully migrating the MongoDB collection while keeping the app running. Operational work doesn't make the product better. It just keeps it from getting worse.

When you commit to Postgres as your single database, you get an operational dividend that compounds over time. This chapter is about that dividend.

## One Thing to Monitor

Every database you add to your stack is a monitoring surface. Redis has its own metrics — memory usage, eviction rate, connected clients, command rate, keyspace hits. Elasticsearch has JVM heap, shard allocation, index size, query latency, and node health. Kafka has consumer lag, partition leadership, and throughput per topic. MongoDB has its own oplog, index build progress, and lock metrics.

Learning these metrics, setting up the dashboards, calibrating the thresholds for alerts, and knowing what actions to take when things look wrong — this is non-trivial knowledge. It takes time to build. It requires someone on your team to carry it.

With Postgres as your single database, you have one monitoring surface. One set of metrics to understand deeply. One dashboard. One alert runbook. The `pg_stat_*` views, `EXPLAIN ANALYZE`, `pg_stat_statements` — these are the instruments, and there are books (this one) and careers built around understanding them.

Depth of understanding is not equally valuable on every system. Understanding Postgres deeply is worth more than understanding six different databases shallowly. The team with one database and deep expertise in it is more reliable than the team with six databases and shallow expertise in all of them.

## One Thing to Back Up

Every database has a backup story. That story includes:
- Which tool to use (pg_dump? mongodump? elasticdump?)
- Where backups go (S3? local disk? backup service?)
- How to test that a backup is good (by trying to restore it)
- What the recovery procedure is (documented somewhere?)
- Who knows how to do it under pressure at 3am

Multiply this by the number of databases in your stack. In practice, most teams have a reasonable backup story for their primary relational database, and a confused or absent backup story for everything else. "We can restore the Postgres database" is reassuring; "we can restore the Postgres database, but we're not sure about the Elasticsearch index, and the Redis cluster has no WAL archiving" is not.

A Postgres-only stack means one backup story, executed well. WAL archiving to S3, base backups via pgBackRest, retention policies, and tested restores. This is achievable for any team. Doing it for five different databases is not.

## One Thing to Understand

The complexity of a system is roughly proportional to the number of concepts it asks you to hold simultaneously. Each database brings its own concepts:

- Redis: RESP protocol, eviction policies, persistence modes (RDB, AOF), cluster mode, Sentinel
- Elasticsearch: Lucene segments, shard allocation, index lifecycle management, mapping types, cluster state
- MongoDB: replica set oplog, sharding, document validation, aggregation pipeline, read preferences
- Kafka: partitions, consumer groups, offsets, compaction, ISR, ZooKeeper/KRaft

A new engineer joining a team with all of these in the stack faces a months-long onboarding process before they can contribute confidently. Not because any one database is impenetrable, but because the aggregate is a lot. And the more senior engineers spend time mentoring that onboarding, the less time they spend building.

With Postgres, the learning curve is real but finite. SQL is universal. The query planner, MVCC, EXPLAIN ANALYZE, WAL, autovacuum — these are learnable, well-documented, and transferable. An engineer who understands Postgres deeply can work effectively on any Postgres-backed system. That's not true for a polyglot stack.

## One Deployment to Update

Every database version in your stack is a version to track. When a critical security vulnerability is announced in Redis, you have to update Redis. When Elasticsearch releases a major version, you have to evaluate and execute the migration. When MongoDB changes its licensing, you have to evaluate alternatives. When Kafka's coordinator migration is required for a new version, you have to plan the maintenance.

These are not optional. Security vulnerabilities get exploited. Outdated software accrues technical debt. Vendor support ends. The more databases you run, the more version management overhead you carry, indefinitely.

Postgres has a major version release roughly once a year, with a 5-year support window. There is one vendor (the PostgreSQL Global Development Group), one community, and an exceptionally stable upgrade path. `pg_upgrade` handles major version upgrades with minimal downtime. The upgrade discipline for one database is manageable.

## One Transaction Boundary

The most underappreciated benefit of a single database: transactional consistency across all your data.

Consider a common operation: a user purchases a product, which should:
1. Decrement the product inventory
2. Create an order record
3. Enqueue an order confirmation email
4. Update the user's last_purchase_at timestamp
5. Record an audit event
6. Award loyalty points

In a polyglot stack:
- Steps 1, 2, 4, 5 are in Postgres
- Step 3 is a Redis queue operation
- Step 6 might be in Redis or Mongo

There is no transaction that spans Postgres and Redis. You can use the Saga pattern, the outbox pattern, or careful application logic to maintain consistency, but you've accepted that these operations can be partially applied. A process crash after step 2 and before step 3 leaves you with an order that never sends a confirmation email. A Redis timeout leaves you with an order that has no confirmation job.

With Postgres handling all of these (jobs in a jobs table, events in an audit table), steps 1-6 are all in a single transaction. Either all happen or none do. No partial state. No compensating transactions. No Saga coordinator. No idempotency keys. This eliminates entire categories of bugs that distributed consistency requires application code to handle.

## The Staffing Argument

The argument for operational simplicity becomes most tangible when you think about staffing.

A startup with five databases needs:
- A backend engineer who understands Postgres
- Someone who understands Redis well enough to tune it when it misbehaves
- Someone who understands Elasticsearch well enough to rebalance shards
- Someone who can handle Kafka consumer lag incidents

In a small team, these roles overlap. The backend engineer is also the Redis person is also the Elasticsearch person. That person is necessarily shallow on all of them. Or you hire specialists — which is expensive, and you have a team full of people who don't fully understand each other's domains.

A startup with one database needs people who understand Postgres. That's a smaller requirement, easier to hire for, and produces a team that can cover for each other.

As you scale, the calculus changes — large organizations can afford deep specialization. But the "move fast" phase of a startup is exactly when you should resist the urge to build a complex data infrastructure. The complexity you introduce early becomes the technical debt you manage forever.

## The Knowledge Half-Life Problem

Technology knowledge has a half-life. The Redis you learned in 2018 is not quite the same as the Redis you need to operate in 2024. The Elasticsearch you understood in 2020 has changed substantially. MongoDB Atlas has different operational characteristics than self-hosted MongoDB.

As the databases in your stack change, the knowledge of how to operate them depreciates. Your team's operational expertise becomes stale, and the gaps become risks.

Postgres's architecture is remarkably stable. The concepts you learn today — MVCC, WAL, VACUUM, the query planner, the extension system — will be relevant in ten years. The investment in Postgres expertise has a longer half-life than most. That makes it more valuable per hour of learning invested.

## What Operational Simplicity Actually Looks Like

Here is what a team running Postgres-only actually experiences differently:

**Incident response:** An alert fires. The on-call engineer opens `pg_stat_activity`, `pg_stat_statements`, and the Grafana dashboard. All the relevant data is in one place. The mental model they need is one they've built over time.

**Onboarding:** A new engineer joins. You point them to the database. They can run `\dt` and see everything. The schema is visible and queryable. There's no "but the search index is over here and the job queue is over here" orientation session.

**Capacity planning:** You need to know if you can handle twice the traffic. You look at query times, index sizes, and cache hit rates — all in one system. You can project with reasonable confidence.

**Disaster recovery:** Something goes catastrophically wrong. You restore from backup. One backup, one restore procedure, one system to verify. You've tested it. You're confident.

**Debugging a production bug:** A record is in an unexpected state. You can query the audit log, the job history, and the data — all with SQL, all in one transaction boundary, with consistent timestamps and IDs.

## When the Simplicity Argument Breaks Down

Operational simplicity is compelling, but it's not infinite. There's a scale at which the "one database" model genuinely creates problems:

- When your database is so large that read replicas can't keep up with reporting queries and you need a data warehouse
- When your write throughput exceeds what a single Postgres instance can handle
- When your team is large enough that multiple teams need isolation from each other's schemas
- When you have regulatory requirements that mandate different storage for different data types

These thresholds are real, but they're typically reached at a scale larger than most teams realize. The team that adds a data warehouse when they have 10 million rows is over-engineering. The team that adds Redis for rate limiting before profiling whether Postgres can handle it is over-engineering.

The principle is: default to operational simplicity, add complexity only when the evidence demands it, and be honest about what the evidence actually says.

## The Compounding Effect

The benefits of operational simplicity compound. The engineer who avoids three database-specific rabbit holes in their first year spends that time building product. The team that doesn't debug a Kafka consumer lag incident doesn't get paged at 2am. The startup that ships with one database has a simpler stack to migrate when requirements change.

These are not dramatic wins in any individual moment. They're small, consistent advantages that accumulate into a significant competitive edge over time. Operational simplicity is not glamorous, but it is one of the most durable engineering advantages a team can have.

Postgres is enough. The operability dividend is a large part of why.
