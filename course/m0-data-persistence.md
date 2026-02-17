# Data & Persistence

PostgreSQL configuration, environment variables, and connection pooling were covered in the previous chapters. This chapter focuses on what you need to keep that data **safe and correctly structured** — backups, migrations, and how multi-tenancy works under the hood.

---

## Backup Strategies

### What to Back Up

| Item | Priority | Method |
|------|----------|--------|
| Application Database (PG_DB) | **Critical** | pg_dump or managed snapshots |
| ToolJet Database (TOOLJET_DB) | **Critical** (if used) | pg_dump or managed snapshots |
| `LOCKBOX_MASTER_KEY` | **Critical** | Secure key management (Vault, AWS Secrets Manager, etc.) |
| `SECRET_KEY_BASE` | **Critical** | Same as above |
| `PGRST_JWT_SECRET` | **Critical** (if using TJ DB) | Same as above |
| Environment variables / .env | Important | Version control or secrets manager |
| Redis data | Low | Transient by design; jobs can be re-queued |

:::warning
**The `LOCKBOX_MASTER_KEY` is not recoverable.** If you lose it, all encrypted datasource credentials become unreadable. You would need to re-enter every datasource connection detail. Treat this key like a database encryption key — back it up in a separate, secure location.
:::

### Managed PostgreSQL (RDS, Cloud SQL, Azure Database)

- Enable automated daily snapshots with a retention period of at least 7 days
- Enable point-in-time recovery (PITR) — this uses WAL archiving and lets you restore to any second within the retention window
- Test restores quarterly

### Self-Managed PostgreSQL

Schedule `pg_dump` via cron for logical backups:

```bash
# Daily logical backup, retain 7 days
pg_dump -h localhost -U tooljet -Fc tooljet > /backups/tooljet_$(date +%Y%m%d).dump
pg_dump -h localhost -U tooljet -Fc tooljet_db > /backups/tooljet_db_$(date +%Y%m%d).dump
find /backups -name "*.dump" -mtime +7 -delete
```

For large databases, use `pg_dump -Fd -j 4` (directory format with 4 parallel workers) to reduce backup time.

Set up WAL archiving for PITR if your data is critical.

### What About Redis?

Redis data in ToolJet is transient — job queues and pub/sub channels. If Redis goes down, pending jobs are lost, but the system recovers on restart. Scheduled workflows re-populate from the database. Don't over-invest in Redis backup unless you need zero-loss guarantees on in-flight workflow executions.

---

## Migration Considerations

### How Migrations Work

ToolJet uses TypeORM migrations with these key properties:

- **`migrationsRun: false`** — Migrations don't run automatically on server start
- **`synchronize: false`** — TypeORM never auto-modifies the schema
- **`migrationsTransactionMode: 'all'`** — Each migration runs in a single transaction (all-or-nothing)

You run migrations explicitly:

```bash
# Development
npm run db:migrate

# Production
npm run db:migrate:prod
```

The migrate command runs two sets sequentially:
1. **Schema migrations** — DDL changes (create/alter tables, indexes)
2. **Data migrations** — DML changes (backfill data, transform existing rows)

### Upgrade Workflow

When upgrading ToolJet to a new version:

1. **Back up both databases** before doing anything
2. **Stop ToolJet server and workers** — never run migrations while the app is serving traffic
3. **Run migrations** against the new version's code
4. **Start the new version**

Failed migrations roll back cleanly thanks to the transaction wrapping. But backups are still your safety net — data migrations that transform rows can't be "un-transformed" after a commit.

### Migration Lock

ToolJet includes `LockMigrationsTable1` and `LockMigrationsTable2` migration files to prevent concurrent migration runs. If you're running multiple instances in Kubernetes and they all try to migrate on startup, the lock prevents corruption. Only one process runs migrations; the others wait or skip.

### Zero-Downtime Upgrades

ToolJet does not natively support zero-downtime schema migrations. The recommended approach:

1. Scale down to 1 replica
2. Run migrations
3. Start the new version
4. Scale back up

For truly zero-downtime deployments, you'd need a blue-green strategy: keep the old version running, migrate a parallel database, switch traffic. This is complex and rarely necessary for internal tooling platforms.

---

## Multi-Tenancy Model

ToolJet uses a **single-database, shared-schema** multi-tenancy model.

### How It Works

- The tenant boundary is the **Organization** (called "Workspace" in the UI)
- Every major entity — apps, datasources, users, environments, permissions — has an `organization_id` foreign key
- All organizations share the same database tables
- Data isolation is enforced at the **application layer**, not the database layer

### What This Means in Practice

**Advantages:**
- Simple to operate — one database to back up, monitor, and maintain
- Migrations run once and apply to all organizations
- No per-tenant database provisioning needed

**Things to be aware of:**
- There's no row-level security (RLS) at the PostgreSQL level. A bug in the application layer could theoretically leak data across organizations. This is standard for most multi-tenant SaaS platforms, but worth noting for compliance conversations.
- A heavy user in one organization (e.g., running expensive queries, creating thousands of apps) affects database performance for all organizations. There's no per-tenant resource isolation at the database level.
- If you need hard tenant isolation (separate databases per tenant), ToolJet doesn't support that out of the box. You'd need separate ToolJet deployments.

### The ToolJet Database Exception

When users create tables via the ToolJet Database UI, those tables are created in the `TOOLJET_DB` database with organization-scoped naming. PostgREST serves as the API gateway, with JWT-based access control tying requests to the correct organization.

This means the `TOOLJET_DB` database can grow unpredictably — it depends entirely on how your users use the feature. Monitor its size separately from the application database.

---

## Quick Checklist

Before going to production:

- [ ] `LOCKBOX_MASTER_KEY`, `SECRET_KEY_BASE`, and `PGRST_JWT_SECRET` backed up securely
- [ ] Automated database backups configured with PITR enabled
- [ ] You've tested a restore at least once
- [ ] You have a documented migration runbook for upgrades

---

## What's Next

Next up: **Observability Baseline** — what to monitor, which health endpoints to hit, log aggregation setup, and alerting recommendations.
