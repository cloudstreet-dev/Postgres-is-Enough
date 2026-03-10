# Replication and High Availability

A single Postgres instance, no matter how well-tuned, is a single point of failure. High availability means the database continues to serve traffic when a hardware failure, network partition, or software crash takes down the primary server. Replication is the mechanism that makes this possible.

Postgres's replication story is mature, well-understood, and capable of supporting serious production requirements — from simple read replicas to automatic failover to logical change data capture.

## Physical (Streaming) Replication

Streaming replication is the primary replication mechanism in Postgres. The standby connects to the primary, streams WAL records, and applies them to maintain a byte-for-byte copy of the primary's data.

**How it works:**
1. The primary writes all changes to WAL (Write-Ahead Log)
2. The standby's WAL receiver process connects to the primary's WAL sender process
3. WAL records stream continuously to the standby
4. The standby's startup process applies WAL records, keeping the standby's data in sync
5. The standby is read-only — queries can run against it, but no writes are allowed

### Setting Up Streaming Replication

On the primary, in `postgresql.conf`:

```ini
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB  # Keep enough WAL for standbys to catch up
```

Create a replication role:

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secure_password';
```

In `pg_hba.conf`, allow the standby to connect for replication:

```
host    replication   replicator   10.0.0.2/32   scram-sha-256
```

On the standby, create a `standby.signal` file and configure `postgresql.conf`:

```ini
primary_conninfo = 'host=10.0.0.1 port=5432 user=replicator password=secure_password'
restore_command = ''  # Only needed for WAL archiving
```

The standby automatically starts replicating when it finds `standby.signal` in the data directory.

Verify replication is working on the primary:

```sql
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
```

Replication lag is shown in `replay_lag` — the time difference between when the primary committed a transaction and when the standby applied it.

### Synchronous vs. Asynchronous Replication

**Asynchronous replication (default):** The primary commits and returns to the client without waiting for the standby to acknowledge. The standby eventually applies the changes. Replication lag is typically milliseconds to seconds. If the primary crashes, recent transactions may not have reached the standby — *replication lag = potential data loss*.

**Synchronous replication:** The primary waits for one or more standbys to acknowledge writing the WAL before committing. Zero data loss on primary failure, at the cost of added commit latency (every commit waits for network round-trip to standby).

Configure synchronous replication on the primary:

```ini
synchronous_standby_names = 'standby1'  # Specific standby name
# Or: 'ANY 1 (standby1, standby2)'  # Any 1 of multiple standbys
```

On the standby, set `application_name` in `primary_conninfo`:

```ini
primary_conninfo = 'host=primary port=5432 user=replicator application_name=standby1'
```

For most applications, asynchronous replication with a well-monitored replica is sufficient. The typical replication lag of milliseconds means RPO (Recovery Point Objective) is measured in seconds, not minutes.

### Replication Slots

A replication slot guarantees that the primary retains WAL segments until all subscribers have consumed them. Without a slot, if a standby falls behind (network outage, maintenance), the primary might delete WAL that the standby still needs, requiring a full resync.

```sql
-- Create a physical replication slot
SELECT pg_create_physical_replication_slot('standby1_slot');

-- List slots and their lag
SELECT slot_name, active, pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
FROM pg_replication_slots;
```

**Warning:** Replication slots that fall behind and stop consuming WAL will cause WAL to accumulate indefinitely, eventually filling the disk. Monitor slot lag and set `max_slot_wal_keep_size` as a safety valve:

```ini
max_slot_wal_keep_size = 10GB  # Discard WAL for lagging slots beyond this
```

## Read Replicas

A standby server in hot standby mode accepts read-only queries. This is a simple way to scale read traffic.

```sql
-- On primary: hot standby is the default
-- On standby, queries run normally (read-only)
SELECT COUNT(*) FROM orders;  -- works on standby

-- Queries requiring writes fail:
INSERT INTO orders ...;  -- ERROR: cannot execute INSERT in a read-only transaction
```

Read replica use cases:
- Long-running analytical queries that would impact primary performance
- Reporting and BI tools
- Full-text search queries
- Geographic distribution (read from a nearby region)

For application routing, you need to direct read traffic to replicas and write traffic to the primary. Tools like HAProxy, PgBouncer, or application-level connection routing handle this.

**Standby hint bits:** One subtlety — when a read query on the standby touches a tuple that needs its hint bits set (a tuple that hasn't been frozen), the standby can't update the hint bits (it's read-only). Postgres handles this gracefully (it applies the visibility rules manually), but it means hot standby reads can be slightly slower than primary reads for data that's never been frozen.

## Logical Replication

Physical replication creates an exact byte-for-byte copy of the primary. Logical replication replicates changes at the row level — individual INSERT, UPDATE, DELETE events — allowing more flexibility:

- Replicate a subset of tables
- Replicate to a different major Postgres version
- Replicate to multiple downstream consumers
- Support bidirectional replication (with care)
- Use as a CDC (Change Data Capture) source

Logical replication uses a publish/subscribe model:

**Publisher (primary):**
```sql
-- Requires wal_level = logical in postgresql.conf
CREATE PUBLICATION my_publication
    FOR TABLE users, orders, products;
-- Or: FOR ALL TABLES (replicates everything)
```

**Subscriber (downstream Postgres):**
```sql
CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=primary port=5432 user=replicator password=xxx dbname=mydb'
    PUBLICATION my_publication;
```

The subscription starts copying the initial data, then applies ongoing changes. Tables on the subscriber must exist with compatible schemas.

### Logical Decoding and CDC

Logical replication is built on *logical decoding* — the ability to decode WAL into a stream of data changes. Tools like Debezium use logical decoding to stream Postgres changes into Kafka, enabling event sourcing and data integration patterns without polling:

```sql
-- Create a logical replication slot for an external consumer
SELECT pg_create_logical_replication_slot('debezium', 'pgoutput');

-- Peek at changes (for debugging)
SELECT * FROM pg_logical_slot_peek_changes('debezium', NULL, NULL);
```

Debezium + Kafka is the gold standard for streaming Postgres changes to other systems. This is how you bridge Postgres (the source of truth) with Elasticsearch, Redis, data warehouses, and other consumers while maintaining Postgres as the authoritative store.

## Automatic Failover with Patroni

Physical replication sets up the data flow, but failover — promoting a standby when the primary fails — requires additional tooling. Patroni is the most widely-used solution.

**Patroni** is a Postgres cluster manager that:
- Uses a distributed consensus store (etcd, Consul, or ZooKeeper) to elect a leader
- Automatically promotes a standby when the primary becomes unavailable
- Manages the Postgres configuration for the current role (primary vs. standby)
- Provides a REST API for cluster status and management

A Patroni cluster consists of:
- 2 or more Postgres instances with Patroni agents
- A distributed consensus store (3-node etcd cluster is typical)
- (Optionally) HAProxy or similar for client routing

**Basic Patroni configuration** (`patroni.yml`):

```yaml
name: postgres1
scope: my-cluster

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.0.1:8008

etcd3:
  hosts: 10.0.0.10:2379,10.0.0.11:2379,10.0.0.12:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB - max allowed lag for a standby to be eligible

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.0.1:5432
  data_dir: /var/lib/postgresql/data
  parameters:
    wal_level: replica
    hot_standby: on
    max_wal_senders: 10
    max_replication_slots: 10
```

Patroni handles:
- Leader election (which Postgres node is primary)
- Automatic failover (promotes best standby when primary fails)
- Replication configuration management
- Health checking

**The patronictl CLI** for cluster management:
```bash
patronictl -c /etc/patroni.yml list          # Show cluster state
patronictl -c /etc/patroni.yml failover my-cluster  # Manual failover
patronictl -c /etc/patroni.yml switchover my-cluster  # Planned failover
```

## Client Routing

When the primary changes (failover), your application needs to know the new primary's address. Options:

**DNS-based routing:** Maintain a DNS record that always points to the current primary. Patroni updates this record on failover. Simple but has TTL lag.

**HAProxy:** A reverse proxy that routes connections based on health checks. Patroni's REST API reports whether a node is primary or standby. HAProxy queries this to route traffic.

```
frontend postgres_primary
    bind *:5432
    default_backend primary

backend primary
    option httpchk GET /primary
    server postgres1 10.0.0.1:5432 check port 8008
    server postgres2 10.0.0.2:5432 check port 8008
```

**PgBouncer + Patroni:** PgBouncer can be reconfigured to point to the new primary after failover, either via a patroni callback or by pointing to a virtual IP that moves with the primary.

**Cluster-aware drivers:** Some Postgres drivers support multiple hosts and automatically discover the current primary. For example, in Go with `pgx`:

```go
connStr := "postgres://user:pass@host1,host2,host3/mydb?target_session_attrs=read-write"
```

This connects to the first host in the list that accepts read-write connections (i.e., the primary).

## Replication Monitoring

Essential metrics to monitor:

```sql
-- Replication lag on primary
SELECT client_addr, replay_lag, sync_state
FROM pg_stat_replication;

-- On standby: how far behind is this standby?
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;

-- WAL sender activity
SELECT pid, state, sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;
```

Alert on:
- Replication lag > 60 seconds (normal is <100ms for async)
- Replication slot lag growing indefinitely
- Standby not connected at all

## Cloud Managed HA

If you're using a managed Postgres service (AWS RDS, Cloud SQL, Aurora, Supabase), high availability is typically handled for you:

- **AWS RDS Multi-AZ:** Synchronous standby in a different availability zone. Automatic failover in 60-120 seconds. Read replicas available separately.
- **AWS Aurora:** Multi-AZ by design with storage-level replication. Very fast failover (~30 seconds). Up to 15 read replicas.
- **Google Cloud SQL:** HA with an automatic standby in a different zone. Failover in ~60 seconds.
- **Supabase:** Built-in HA with read replicas.

Managed HA trades control for operational simplicity. For teams without dedicated DBAs, managed HA is often the right choice — let the cloud provider handle the Patroni equivalent.

## The Availability Math

A single Postgres instance with good hardware and no HA has roughly 99.9% availability (about 8 hours of downtime per year from scheduled maintenance and unexpected failures). With HA and automatic failover, you can achieve 99.95% or better (under 5 hours per year). The gap between "I should add a standby" and "this will cause production downtime" is often just one hardware failure away.

For any production system with users, HA is not optional — it's part of operating responsibly. The complexity of Patroni (or using a managed service) is a one-time investment that pays off the first time a disk fails or a host becomes unavailable and traffic just... fails over.
