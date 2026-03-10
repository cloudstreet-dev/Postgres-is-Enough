# Security

PostgreSQL has one of the most sophisticated security models of any open-source database. Roles, row-level security, column-level permissions, encryption, audit logging — the tools are all there. The failure mode is not missing features; it's not using them.

This chapter covers how to use Postgres's security features to build a database that's properly defended: least-privilege access, data-level access control, encrypted connections and storage, and comprehensive audit trails.

## The Role System

Postgres security is organized around *roles*. A role is a named identity in the database that can hold privileges, own objects, and log in (or not). The distinction between "users" (roles that can log in) and "groups" (roles that can't log in but grant privileges) is a convention, not a hard distinction.

### Creating Roles

```sql
-- A login role (user)
CREATE ROLE app_user WITH LOGIN PASSWORD 'secure_password';

-- A login role with expiry
CREATE ROLE temp_analyst WITH LOGIN PASSWORD 'password'
    VALID UNTIL '2025-01-01';

-- A non-login role (group)
CREATE ROLE readonly;
CREATE ROLE readwrite;
CREATE ROLE admin;
```

### Granting Privileges

```sql
-- Grant schema usage (required to access objects within the schema)
GRANT USAGE ON SCHEMA public TO readonly;
GRANT USAGE ON SCHEMA public TO readwrite;

-- Grant SELECT on all existing tables
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

-- Grant SELECT/INSERT/UPDATE/DELETE on all existing tables
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite;

-- Grant sequence usage (for INSERT with serial/identity columns)
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO readwrite;

-- Default privileges for future objects
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO readonly;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO readwrite;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO readwrite;

-- Grant role membership (app_user gets readonly privileges)
GRANT readonly TO app_user;
```

The `ALTER DEFAULT PRIVILEGES` command is critical — without it, new tables created in the future won't automatically grant the expected privileges to existing roles.

### The Principle of Least Privilege

Every application connection should use a role with only the privileges it needs:

```sql
-- Application: reads and writes data, but never drops tables
CREATE ROLE myapp_user WITH LOGIN PASSWORD 'app_password';
GRANT readwrite TO myapp_user;

-- Read replica: SELECT only
CREATE ROLE myapp_reader WITH LOGIN PASSWORD 'reader_password';
GRANT readonly TO myapp_reader;

-- Migrations: need to create/alter tables
CREATE ROLE myapp_migrations WITH LOGIN PASSWORD 'migration_password';
GRANT readwrite TO myapp_migrations;
-- Also grant CREATE privileges:
GRANT CREATE ON SCHEMA public TO myapp_migrations;

-- Admin: SUPERUSER or carefully-scoped privileges
-- Prefer specific privileges over SUPERUSER in production
```

Separate credentials for migrations (run during deployments) vs. application connections (run all the time) is a security hygiene practice with real value: if the application credential is compromised, the attacker can't run DDL.

## `pg_hba.conf`: Client Authentication

`pg_hba.conf` (host-based authentication) controls which hosts can connect to which databases with which roles and which authentication methods.

```
# TYPE  DATABASE  USER     ADDRESS          METHOD
local   all       all                       peer         # OS user = DB user
host    all       all      127.0.0.1/32     scram-sha-256
host    mydb      app_user 10.0.1.0/24      scram-sha-256
hostssl all       all      0.0.0.0/0        scram-sha-256  # Require SSL
hostnossl all     all      0.0.0.0/0        reject        # Reject non-SSL
```

Key authentication methods:
- `peer`: Use OS username (local connections only). The OS user must match the database role name.
- `scram-sha-256`: Password authentication using SCRAM. Modern and secure. Prefer over `md5`.
- `md5`: Password authentication using MD5. Deprecated — use `scram-sha-256` instead.
- `cert`: Client certificate authentication. Most secure for automated connections.
- `reject`: Deny the connection outright.

**Always use `scram-sha-256` or `cert`, never `md5` or `trust` in production.**

`trust` means any connection from the matched host/user is allowed without a password. Appropriate only for local Unix socket connections on tightly controlled systems, and even then, risky.

## SSL/TLS

All external connections should use SSL/TLS to encrypt data in transit. Any credential or sensitive data query over an unencrypted connection can be sniffed.

In `postgresql.conf`:
```ini
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'ca.crt'  # For client certificate verification
ssl_min_protocol_version = 'TLSv1.2'
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'
```

Force SSL in `pg_hba.conf` by using `hostssl` entries (only allow SSL) and `hostnossl ... reject` (reject non-SSL):
```
hostssl all all 0.0.0.0/0 scram-sha-256
hostnossl all all 0.0.0.0/0 reject
```

In your application connection string, include `sslmode=verify-full` (or at minimum `sslmode=require`) to verify the server certificate.

## Row-Level Security (RLS)

Row-Level Security is Postgres's mechanism for restricting which rows a role can see or modify. It's declarative, transparent to the application, and enforced at the database layer regardless of how the query arrives.

Enable RLS on a table:
```sql
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
```

Create a policy:
```sql
-- Users can only see their own orders
CREATE POLICY orders_user_isolation ON orders
    FOR ALL
    TO app_user
    USING (user_id = current_setting('app.current_user_id')::bigint);
```

The `USING` clause is checked for SELECT, UPDATE, and DELETE (which rows can you read/modify?). The `WITH CHECK` clause is checked for INSERT and UPDATE (which values can you write?):

```sql
-- Users can only insert orders for themselves
CREATE POLICY orders_insert_policy ON orders
    FOR INSERT
    TO app_user
    WITH CHECK (user_id = current_setting('app.current_user_id')::bigint);
```

### Setting RLS Context

The application must set the context before executing queries:

```sql
-- At the start of each request/transaction:
SET LOCAL app.current_user_id = '42';
-- Now all queries on `orders` only return rows for user_id = 42
SELECT * FROM orders;  -- Only returns user 42's orders
```

`SET LOCAL` is scoped to the current transaction — it resets when the transaction ends. Safe for use in connection-pooled environments (transaction mode).

### Multi-tenant RLS

RLS is the foundation of multi-tenant data isolation in a shared database:

```sql
-- Tenant isolation across all tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.tenant_id')::bigint);

CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::bigint);

-- Admin role bypasses RLS
CREATE ROLE admin_user WITH LOGIN;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;  -- Applies RLS even to table owner
GRANT BYPASS RLS ON orders TO admin_user;  -- Explicitly grant bypass when needed
```

Note: `FORCE ROW LEVEL SECURITY` applies RLS even to the table owner. Without it, the table owner bypasses RLS. In multi-tenant systems, you typically want `FORCE ROW LEVEL SECURITY` to prevent ownership-based bypass.

### RLS Performance

RLS policies add a predicate to every query. For policies on indexed columns (like `tenant_id`), the query planner can use the index, and performance impact is minimal. For policies on unindexed columns or complex expressions, every query pays the cost of evaluating the policy.

Index the columns used in RLS policies:
```sql
CREATE INDEX idx_orders_tenant_id ON orders(tenant_id);
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

## Column-Level Security

Restrict access to specific columns using column-level privileges:

```sql
-- Grant SELECT on all columns except ssn and salary
GRANT SELECT (id, name, email, created_at) ON employees TO hr_viewer;

-- Or: grant all columns, then revoke the sensitive ones
GRANT SELECT ON employees TO hr_viewer;
REVOKE SELECT (ssn, salary) ON employees FROM hr_viewer;
```

Column-level security is useful for compliance requirements where certain columns (PII, financial data) must be visible only to specific roles.

## Encryption at Rest

Postgres itself doesn't encrypt data files. Encryption at rest is typically handled at the storage level:

- **OS-level:** Linux LUKS (dm-crypt) encrypts the entire filesystem, including Postgres data files
- **Cloud:** AWS RDS encrypts with AWS KMS, GCS with Cloud KMS — this is typically enabled by default
- **Tablespace-level:** Some storage systems provide per-volume encryption
- **pgcrypto:** Column-level encryption within the database (see below)

For cloud databases, ensure encryption at rest is enabled (it usually is by default).

## Column-Level Encryption with pgcrypto

For particularly sensitive data that should be encrypted even from database administrators:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Store an encrypted value
INSERT INTO medical_records (patient_id, diagnosis_encrypted)
VALUES (1, pgp_sym_encrypt('Type 2 Diabetes', 'encryption_key'));

-- Retrieve and decrypt
SELECT patient_id, pgp_sym_decrypt(diagnosis_encrypted, 'encryption_key')
FROM medical_records
WHERE patient_id = 1;
```

For public-key encryption (where different parties encrypt vs. decrypt):
```sql
-- Encrypt with public key
UPDATE users
SET ssn_encrypted = pgp_pub_encrypt(ssn, dearmor('-----BEGIN PGP PUBLIC KEY...'))
WHERE id = 1;

-- Decrypt requires private key (held outside the DB)
SELECT pgp_pub_decrypt(ssn_encrypted, dearmor('-----BEGIN PGP PRIVATE KEY...'), 'passphrase')
FROM users WHERE id = 1;
```

Column-level encryption has real costs: the data is not indexable, not queryable by value, and requires the application to manage keys. Use it only where the compliance requirement specifically demands it.

## Audit Logging

For compliance and security incident investigation, knowing who did what and when is essential.

### pgaudit

The `pgaudit` extension (Chapter 11) provides statement-level and object-level audit logging. Configure it in `postgresql.conf`:

```ini
shared_preload_libraries = 'pgaudit'

# Log all write operations and DDL
pgaudit.log = 'write, ddl'

# Include connection info in log
pgaudit.log_client = on

# Log role that granted the privilege being used
pgaudit.log_relation = on
```

Per-role audit configuration:
```sql
-- Audit all operations by this user
ALTER ROLE sensitive_user SET pgaudit.log = 'all';
```

### Built-in Logging

Postgres's standard logging can also provide audit trails:

```ini
log_connections = on
log_disconnections = on
log_duration = on
log_statement = 'mod'  # Log all DML; or 'all' for everything
log_min_duration_statement = 0  # Log all statements (combine with log_statement)
```

Log everything to a table via `log_destination = 'csvlog'` and import into a log analysis system.

### Trigger-Based Audit Tables

For high-fidelity row-level audit trails:

```sql
CREATE TABLE audit_log (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name TEXT NOT NULL,
    operation TEXT NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    row_id BIGINT,
    old_row JSONB,
    new_row JSONB,
    changed_by TEXT NOT NULL DEFAULT current_user,
    changed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log(table_name, operation, row_id, old_row, new_row)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        CASE WHEN TG_OP = 'DELETE' THEN OLD.id ELSE NEW.id END,
        CASE WHEN TG_OP != 'INSERT' THEN row_to_json(OLD)::jsonb END,
        CASE WHEN TG_OP != 'DELETE' THEN row_to_json(NEW)::jsonb END
    );
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

This captures a complete before/after picture for every row change. Partition the `audit_log` table by date for manageability.

## Security Checklist

A practical checklist for a production Postgres deployment:

- [ ] All external connections use SSL/TLS (`hostssl` in pg_hba.conf)
- [ ] Password authentication uses `scram-sha-256`, not `md5` or `trust`
- [ ] Application uses a role with only necessary privileges (not superuser)
- [ ] Separate credentials for migration runner vs. application
- [ ] `pg_hba.conf` limits which hosts can connect
- [ ] Default passwords changed (especially `postgres` role)
- [ ] RLS enabled for multi-tenant or user-scoped tables
- [ ] Column-level grants for sensitive data (SSN, salary, health data)
- [ ] `pgaudit` or trigger-based audit logging for compliance requirements
- [ ] Encryption at rest enabled (storage-level or column-level for most-sensitive data)
- [ ] Postgres data directory accessible only to the `postgres` OS user
- [ ] `pg_hba.conf` and `postgresql.conf` not world-readable

PostgreSQL provides all the mechanisms needed for a properly secured database. The work is in applying them consistently. The most common security failures are not clever exploits — they're misconfigured authentication, overly privileged application accounts, and missing audit trails for compliance requirements. The checklist above addresses all of them.
