# Backup and Recovery

A database you can't restore is a database you don't have. Backup is not about the backup — it's about the restore. The question is not "do we have backups?" but "have we tested restoring from them recently?" and "what's our actual recovery time in an emergency?"

This chapter covers the full spectrum of Postgres backup strategies: logical dumps with `pg_dump`, continuous archiving with WAL, point-in-time recovery, and the production-grade tools that make this manageable.

## Terminology: RPO and RTO

Two metrics define your backup requirements:

**RPO (Recovery Point Objective):** How much data can you afford to lose? An RPO of 5 minutes means you can tolerate losing up to 5 minutes of transactions in a catastrophic failure. An RPO of 0 means you need synchronous replication with no data loss.

**RTO (Recovery Time Objective):** How long can the database be unavailable during recovery? An RTO of 4 hours means you can take up to 4 hours to restore from a backup and catch up. An RTO of 5 minutes requires a hot standby.

Your backup strategy flows from these numbers. A startup with moderate data loss tolerance might accept RPO=24h (daily backups) and RTO=4h. A financial system might require RPO=0 and RTO=minutes. Most production applications land somewhere in between.

## pg_dump: Logical Backups

`pg_dump` is Postgres's built-in tool for creating logical (SQL) backups. It connects to the database and dumps the schema and data as SQL commands or CSV.

```bash
# Dump to SQL file (text format)
pg_dump --dbname=mydb --file=backup.sql

# Dump to custom binary format (recommended: smaller, parallelizable restore)
pg_dump --dbname=mydb --format=custom --file=backup.dump

# Dump with connection string
pg_dump "postgresql://user:pass@host:5432/mydb" --format=custom --file=backup.dump

# Dump only specific tables
pg_dump --table=orders --table=users --format=custom --file=partial_backup.dump

# Dump schema only (no data)
pg_dump --schema-only --format=custom --file=schema_only.dump

# Dump data only (no schema)
pg_dump --data-only --format=custom --file=data_only.dump
```

### Restoring with `pg_restore`

```bash
# Restore from custom format
pg_restore --dbname=mydb --jobs=4 backup.dump  # parallel restore with 4 workers

# Restore to a new database
createdb mydb_restored
pg_restore --dbname=mydb_restored --jobs=4 backup.dump

# Restore a specific table
pg_restore --dbname=mydb --table=orders backup.dump

# Restore schema only
pg_restore --schema-only --dbname=mydb backup.dump

# From SQL format:
psql --dbname=mydb < backup.sql
```

### `pg_dumpall`

Dumps the entire PostgreSQL cluster — all databases, global objects (roles, tablespaces):

```bash
pg_dumpall --file=cluster_backup.sql --globals-only  # Just roles/tablespaces
pg_dumpall --file=cluster_backup.sql                 # Everything
```

### Limitations of Logical Backups

- **Slow:** Dumping a large database takes hours; restoring takes longer (must replay all INSERT statements)
- **Point-in-time is limited:** A dump reflects the state at one moment. No PITR without WAL archiving.
- **Not suitable for very large databases:** For databases > several hundred GB, dump/restore is too slow for realistic RTO

For small databases (<50GB), pg_dump with daily automation is often sufficient. For larger databases or stricter RPO/RTO requirements, WAL archiving is necessary.

## WAL Archiving: Continuous Backup

Physical backup (PITR) works by:
1. Taking a base backup (a copy of the data directory)
2. Archiving WAL segments continuously as they're generated
3. At recovery time: restore the base backup, then replay WAL segments up to the desired point in time

This allows recovery to any point in time since the base backup — with granularity limited only by WAL segment size.

### Configuring WAL Archiving

In `postgresql.conf`:

```ini
wal_level = replica     # Minimum for archiving
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'
# Or to S3:
# archive_command = 'aws s3 cp %p s3://my-bucket/wal/%f'
# Or with WAL-G:
# archive_command = 'wal-g wal-push %p'
```

`%p` is the path to the WAL file, `%f` is just the filename. The archive command must return 0 on success and non-zero on failure.

### Taking a Base Backup

```bash
# Simple base backup with pg_basebackup
pg_basebackup \
    --host=localhost \
    --username=replicator \
    --pgdata=/var/lib/postgresql/base_backup \
    --format=tar \
    --gzip \
    --compress=9 \
    --wal-method=stream \
    --progress

# Or to a custom location:
pg_basebackup -h localhost -U replicator -D /backup/base -Ft -z -Xs -P
```

`--wal-method=stream` streams WAL during the backup, ensuring the backup is immediately usable without waiting for WAL segments to be archived.

### Point-in-Time Recovery

To recover to a specific point in time:

1. Restore the base backup to the data directory
2. Configure recovery in `postgresql.conf`:

```ini
restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'
# Or from S3: 'aws s3 cp s3://my-bucket/wal/%f %p'

recovery_target_time = '2024-06-15 14:30:00+00'
recovery_target_action = 'promote'  # Promote to primary after reaching target
```

3. Create `recovery.signal` in the data directory
4. Start Postgres

Postgres will restore WAL segments up to the target time, then promote the instance to read-write mode.

**Recovery targets:**
- `recovery_target_time`: Recover to a specific timestamp
- `recovery_target_lsn`: Recover to a specific WAL position
- `recovery_target_xid`: Recover to after a specific transaction
- `recovery_target_name`: Recover to a named restore point (created with `pg_create_restore_point()`)

PITR is invaluable for "logical corruption" disasters — a bad deployment drops the wrong column, a bug deletes the wrong rows. You can recover the database to just before the accident.

## pgBackRest: The Production Standard

pgBackRest is an enterprise-grade backup tool for Postgres, widely considered the most complete solution available. It handles:

- Full, differential, and incremental backups
- WAL archiving and PITR
- Parallel backup and restore (dramatically faster than `pg_basebackup`)
- Backup catalog management
- Encryption and compression
- Remote repositories (S3, GCS, Azure, SFTP)
- Standby backup (don't stress the primary)

### pgBackRest Configuration

```ini
# /etc/pgbackrest/pgbackrest.conf

[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-diff=7
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=very_secure_passphrase

# Or S3:
# repo1-type=s3
# repo1-s3-bucket=my-postgres-backups
# repo1-s3-region=us-east-1
# repo1-s3-endpoint=s3.amazonaws.com

[main]
pg1-path=/var/lib/postgresql/data
pg1-user=postgres
```

PostgreSQL configuration (for archiving):

```ini
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
```

### pgBackRest Operations

```bash
# Create the stanza (repository initialization)
pgbackrest --stanza=main stanza-create

# Take a full backup
pgbackrest --stanza=main backup --type=full

# Take a differential backup (since last full)
pgbackrest --stanza=main backup --type=diff

# Take an incremental backup (since last backup)
pgbackrest --stanza=main backup --type=incr

# List backups
pgbackrest --stanza=main info

# Restore to latest
pgbackrest --stanza=main restore

# PITR to a specific time
pgbackrest --stanza=main restore --recovery-option="recovery_target_time=2024-06-15 14:30:00"

# Restore to a new path
pgbackrest --stanza=main restore --pg1-path=/var/lib/postgresql/restored
```

### Backup Schedule

A typical schedule:
- Full backup: Weekly (Sunday at 2am)
- Differential backup: Daily (every other day at 2am)
- WAL archiving: Continuous

This gives you the ability to restore to any point in time within the retention window, with a restore time of (base backup restore time) + (WAL replay time since backup). With incremental backups, the WAL replay window is small.

## Barman: Another Production Option

Barman (Backup and Recovery Manager) is another mature backup tool, popular in enterprises. It focuses on remote backup management — Barman runs on a dedicated backup server and fetches backups from Postgres servers.

For teams that want Barman's centralized management model, it's an excellent choice. pgBackRest and Barman have roughly equivalent capabilities; the choice is often based on operational preference.

## Testing Your Backups

Here is a hard rule: **if you haven't tested a restore, you don't have a backup.**

Databases fail in ways that expose backup configuration bugs you didn't know existed. Backup processes fail silently. WAL archiving gets misconfigured. Encryption keys get lost. The restore command in your documentation is wrong.

A practical test regime:

**Monthly:** Restore the latest backup to a test environment. Verify the database starts, data is present, and basic queries work.

**After major configuration changes:** Any time you change the backup configuration, test a restore before assuming it works.

**After Postgres upgrades:** Major version upgrades may require updating backup tool versions. Test the restore process after any upgrade.

**Restore test script:**

```bash
#!/bin/bash
set -e

# Restore to a test instance
pgbackrest --stanza=main restore \
    --pg1-path=/var/lib/postgresql/test_restore \
    --target-exclusive \
    --target=latest

# Start the test instance on a different port
postgres -D /var/lib/postgresql/test_restore -p 5433 &

# Wait for startup
until pg_isready -p 5433; do sleep 1; done

# Verify critical data
psql -p 5433 -d mydb -c "SELECT COUNT(*) FROM orders;" | grep -q "[0-9]"
psql -p 5433 -d mydb -c "SELECT max(created_at) FROM orders;"

# Verify WAL applied (check that recent transactions are present)
psql -p 5433 -d mydb -c "SELECT id FROM orders ORDER BY id DESC LIMIT 1;"

echo "Restore test passed"

# Cleanup
pg_ctl stop -D /var/lib/postgresql/test_restore
rm -rf /var/lib/postgresql/test_restore
```

Run this in CI or as a scheduled job. The 30 minutes this takes each month is cheap compared to discovering your backup is broken during an actual disaster.

## The 3-2-1 Rule

Apply the 3-2-1 backup rule to Postgres:
- **3** copies of the data
- **2** different storage media
- **1** offsite copy

In practice:
- **Copy 1:** The live database
- **Copy 2:** A hot standby (streaming replication)
- **Copy 3:** WAL archives and base backups in cloud object storage (S3, GCS)

The standby gives you fast failover (RTO in minutes). The cloud backup gives you PITR and protection against logical corruption and catastrophic datacenter failures.

## Monitoring Backup Health

Critical metrics to alert on:

```sql
-- Is WAL archiving working?
-- (Check pg_stat_archiver)
SELECT archived_count, last_archived_wal, last_archived_time,
       failed_count, last_failed_wal, last_failed_time
FROM pg_stat_archiver;
```

Alert if:
- `last_archived_time` is more than 5 minutes old
- `failed_count` increased since the last check
- Base backup hasn't run in the configured interval
- Backup storage usage is growing unexpectedly

pgBackRest has built-in `check` and `verify` commands:
```bash
pgbackrest --stanza=main check  # Verify archiving is working
pgbackrest --stanza=main verify  # Verify backup files are intact
```

Backup and recovery is the insurance policy for everything in this book. You can optimize performance, harden security, and design elegant schemas — but without working backups, a single hardware failure or operator error can erase it all. Invest in your backup strategy proportionally to the value of your data. For almost any production system, that means WAL archiving, regular base backups to offsite storage, and tested restores.
