# Resource Planning

You can spin up ToolJet on a single 2 GB VM and it'll work. But "works on my machine" is not a deployment strategy. This chapter helps you understand exactly how much compute, memory, storage, and bandwidth each component needs — and more importantly, *how to think about scaling* as your usage grows.

We'll go component by component, give you concrete numbers, and then tie it all together with sizing profiles and a capacity planning worksheet you can actually use.

---

## What You're Sizing For

Before jumping into numbers, understand the four dimensions of ToolJet resource consumption:

1. **Concurrent app builders** — People actively building apps in the editor (uses multiplayer/collaborative editing, frequent API calls to the server)
2. **Concurrent app users** — People using published apps (lighter server load, but more database queries)
3. **Workflow execution volume** — Background jobs, scheduled triggers, webhook-driven automations
4. **ToolJet Database usage** — If you're using ToolJet's built-in database feature, that's a separate PostgreSQL load

Each of these stresses different components differently. A team of 5 builders with 500 app users has a completely different resource profile than a team of 50 builders with 20 app users.

---

## Component-by-Component Breakdown

### ToolJet Server

The server is a Node.js (NestJS) process that handles everything: API requests, authentication, query execution against your datasources, serving the frontend, and optionally processing background jobs.

**What eats memory:**
- Each active datasource connection (database, API, etc.) consumes memory for connection pooling and query result buffering
- Query results are held in memory during transformation before being sent to the client
- The V8 engine itself has a baseline footprint (~150-200MB idle)
- File operations use streaming (efficient), but temporary spikes happen during uploads

**What eats CPU:**
- Query parsing and variable resolution
- JavaScript/Python transformations on query results
- Serving the frontend (if `SERVE_CLIENT=true`)
- SSL termination (if not offloaded to a load balancer)

| Deployment Scale | CPU | Memory | Notes |
|-----------------|-----|--------|-------|
| Development/Testing | 0.5 vCPU | 512MB | Single user, light usage |
| Small team (≤10 builders, ≤100 users) | 1 vCPU | 1GB | Comfortable headroom |
| Medium team (≤50 builders, ≤500 users) | 2 vCPU | 2GB | This is what the Kubernetes defaults use |
| Large deployment (50+ builders, 1000+ users) | 4 vCPU | 4GB | Consider horizontal scaling at this point |

**Key insight:** ToolJet's server is single-threaded (Node.js). Throwing 8 CPUs at a single instance won't help. Once you hit the ceiling of a single process, **scale horizontally** — run multiple server instances behind a load balancer. The Kubernetes deployment defaults to 2 replicas for exactly this reason.

### Workers

Workers are the same ToolJet server binary, started with `WORKER=true`. They don't serve HTTP traffic — they pull jobs from Redis-backed BullMQ queues and execute them.

**What workers process:**
- Workflow executions (manual, webhook-triggered, and scheduled)
- App history snapshots
- Scheduled job orchestration

**What eats memory:**
- Each workflow execution runs JavaScript in an isolated V8 sandbox (default 20MB per isolate via `WORKFLOW_JS_MEMORY_LIMIT_MB`)
- With default concurrency of 5, a single worker can use up to ~100MB just for JS isolates, plus the base process footprint
- Complex workflows with large data payloads increase this further

**What eats CPU:**
- JavaScript execution within workflow nodes
- Data transformations between workflow steps
- Serialization/deserialization of job payloads

| Scenario | CPU | Memory | Concurrency Setting |
|----------|-----|--------|-------------------|
| Light workflows (< 50 executions/hour) | 0.5 vCPU | 512MB | 5 (default) |
| Moderate workflows (50-500/hour) | 1 vCPU | 1GB | 5-10 |
| Heavy workflows (500+/hour) | 2 vCPU | 2GB | 10-15 |
| Burst/mixed workloads | 2 vCPU | 2GB+ | 10+ with multiple replicas |

**When you need a dedicated worker:**
- If you're running workflows at all, a dedicated worker is recommended for production. Without it, workflow processing shares CPU and memory with your HTTP server, meaning a heavy workflow can slow down the app builder UI.
- For single-instance Docker setups without workflows, you can skip the worker entirely.

**Scaling workers:** Workers scale horizontally. Each additional worker replica pulls from the same Redis queue. If your queue depth is growing faster than workers can drain it, add replicas. Monitor via the Bull Board dashboard at `/jobs`.

### PostgreSQL

ToolJet uses **two separate PostgreSQL databases**:

1. **Application Database (PG_DB):** Stores everything — users, organizations, apps, app versions, datasource configs (encrypted), audit logs. This is the core database with 184+ migration tables.
2. **ToolJet Database (TOOLJET_DB):** Only used if you enable the ToolJet Database feature. Stores user-created tables and data. Exposed via PostgREST.

**Application Database sizing:**

| Scale | CPU | Memory | Storage | Max Connections |
|-------|-----|--------|---------|----------------|
| Small (≤10 builders) | 1 vCPU | 1GB | 10GB | 50 |
| Medium (≤50 builders) | 2 vCPU | 2GB | 20GB | 100 |
| Large (50+ builders, many apps) | 4 vCPU | 4-8GB | 50GB+ | 200+ |

**Connection math:** Each ToolJet server instance opens a pool of up to **25 connections** (hardcoded in `ormconfig.ts`). Each database (PG_DB and TOOLJET_DB) gets its own pool. So:

```
Total connections = (server_replicas × 25) + (worker_replicas × 25)
                    × 2 (if using ToolJet Database)
```

For example: 2 server replicas + 1 worker = 3 × 25 = 75 connections for PG_DB alone. If using ToolJet Database, double that to 150. The Helm chart sets PostgreSQL `maxConnections: 1024`, which gives you plenty of room — but if you're using a managed database service (RDS, Cloud SQL), check your instance's connection limit against this formula.

**ToolJet Database sizing:**
- Storage depends entirely on how much data your users put in it
- Statement timeout is 60 seconds by default (`TOOLJET_DB_STATEMENT_TIMEOUT`)
- Bulk uploads capped at 5000 rows / 5MB CSV by default
- If you're not using this feature, you still need the database created, but it'll stay nearly empty

**Storage growth drivers:**
- App versions and history (each save creates version data)
- Audit logs (if enabled)
- Datasource credential storage (encrypted, minimal size)
- ToolJet Database user data (if enabled)

### Redis

Redis serves two purposes in ToolJet:

1. **Job Queue Backend (BullMQ):** Stores job payloads, manages queue state, handles scheduling
2. **Pub/Sub for Collaborative Editing:** Real-time document sync when multiple people edit the same app (via Yjs + ioredis)

**Memory usage:**
- BullMQ stores job data in Redis. With default retention (100 completed + 50 failed jobs), memory usage is modest
- Collaborative editing channels are transient — they exist only while users are actively editing
- Redis memory stays small unless you have massive workflow payloads or thousands of concurrent editors

| Scenario | Memory | Notes |
|----------|--------|-------|
| Single instance, no workflows | 64-128MB | Just multiplayer editing |
| Light workflows | 128-256MB | Small job payloads |
| Heavy workflows with large payloads | 512MB-1GB | Monitor with `INFO memory` |

**Critical configuration:**
- **`maxmemory-policy noeviction`** — BullMQ requires this. If Redis evicts keys, jobs will be silently lost. This is non-negotiable.
- **Persistence (AOF):** Enable `appendonly yes` for durability. Without it, a Redis restart loses all pending jobs.

### When You Need Redis (and When You Don't)

This is one of the most common questions.

**You DO NOT need Redis if:**
- You're running a single ToolJet instance (one container/process)
- You don't use workflows
- You don't need multiplayer/collaborative editing
- You're just evaluating or doing a proof-of-concept

**You DO need Redis if:**
- You run workflows (any kind — manual, scheduled, or webhook)
- You run multiple ToolJet server instances (for HA or scaling)
- You want collaborative editing (multiple builders editing the same app simultaneously)
- You run a separate worker process

In production, you almost always need Redis. The only exception is a truly single-user, single-instance deployment without workflows.

### PostgREST

PostgREST is a lightweight sidecar that provides a REST API over the ToolJet Database. It's only needed if you use the ToolJet Database feature.

| Resource | Value |
|----------|-------|
| CPU | 0.25 vCPU |
| Memory | 128-256MB |
| Image | `postgrest/postgrest:v12.0.2` |

PostgREST is stateless and extremely lightweight. Don't overthink this one.

---

## Sizing Profiles

Here's how it all comes together. These are starting points — monitor and adjust based on actual usage.

### Profile 1: Proof of Concept / Evaluation

**Use case:** Trying out ToolJet, single developer, a few test apps.

| Component | Spec | Notes |
|-----------|------|-------|
| ToolJet Server | 1 vCPU, 1GB RAM | Single instance, `SERVE_CLIENT=true` |
| PostgreSQL | 1 vCPU, 1GB RAM, 10GB disk | Can be on the same machine |
| Redis | Not required | Skip unless testing workflows |
| Worker | Not required | Server handles everything |
| **Total** | **2 vCPU, 2GB RAM** | A single 2GB VM works |

### Profile 2: Small Team Production

**Use case:** 5-15 builders, up to 200 app users, light workflow usage.

| Component | Spec | Notes |
|-----------|------|-------|
| ToolJet Server | 2 vCPU, 2GB RAM | Single instance is fine |
| PostgreSQL | 2 vCPU, 2GB RAM, 20GB disk | Dedicated instance or managed service |
| Redis | 1 vCPU, 512MB RAM | Required for workflows + multiplayer |
| Worker | 1 vCPU, 1GB RAM | Dedicated worker for workflows |
| PostgREST | 0.25 vCPU, 128MB RAM | If using ToolJet Database |
| **Total** | **~6 vCPU, ~6GB RAM** | |

### Profile 3: Mid-size Organization

**Use case:** 20-50 builders, 500+ app users, regular workflow usage, HA required.

| Component | Spec | Notes |
|-----------|------|-------|
| ToolJet Server | 2 × (2 vCPU, 2GB RAM) | Two replicas behind load balancer |
| PostgreSQL | 4 vCPU, 8GB RAM, 50GB disk | Managed service recommended (RDS, Cloud SQL) |
| Redis | 2 vCPU, 1GB RAM | Managed service recommended (ElastiCache, Memorystore) |
| Worker | 2 × (1 vCPU, 1GB RAM) | Two worker replicas |
| PostgREST | 2 × (0.25 vCPU, 128MB) | Match server replica count |
| **Total** | **~14 vCPU, ~16GB RAM** | |

### Profile 4: Large Enterprise

**Use case:** 50+ builders, 1000+ app users, heavy workflows, strict HA/DR requirements.

| Component | Spec | Notes |
|-----------|------|-------|
| ToolJet Server | 3+ × (2 vCPU, 4GB RAM) | HPA with CPU-based autoscaling |
| PostgreSQL | 8 vCPU, 16GB+ RAM, 100GB+ disk | Managed, with read replicas |
| Redis | 4 vCPU, 2GB+ RAM | Managed, with failover |
| Worker | 3+ × (2 vCPU, 2GB RAM) | Scale based on queue depth |
| PostgREST | Match server replicas | |
| **Total** | **30+ vCPU, 40+ GB RAM** | Adjust based on monitoring |

---

## Capacity Planning Worksheet

Use this to calculate your specific requirements. Fill in your numbers:

```
=== YOUR DEPLOYMENT ===

Number of app builders (concurrent):        ____
Number of app users (concurrent):           ____
Number of apps in production:               ____
Workflow executions per hour (estimated):    ____
Using ToolJet Database?                     Yes / No
HA required?                                Yes / No
Target uptime SLA:                          ____

=== COMPUTE ===

Server instances needed:
  - Builders ÷ 15 = minimum replicas (round up)
  - If HA required: minimum 2 replicas
  - Each replica: 2 vCPU, 2GB RAM (adjust if >500 concurrent users)

Worker instances needed:
  - If no workflows: 0
  - If < 200 executions/hour: 1 replica, concurrency 5
  - If 200-1000/hour: 2 replicas, concurrency 10
  - If > 1000/hour: 3+ replicas, concurrency 10-15

=== DATABASE ===

PostgreSQL connections needed:
  - (server_replicas + worker_replicas) × 25 per database
  - × 2 if using ToolJet Database
  - Add 20% headroom
  - Your total: ____

PostgreSQL storage:
  - Base: 5GB (schema + migrations)
  - Per 100 apps: ~2GB (versions, history)
  - ToolJet Database: depends on your data
  - Backups: 2-3x primary storage
  - Your total: ____

=== REDIS ===

Needed?
  - Workflows OR multiple server instances OR multiplayer = Yes
  - Single instance, no workflows, solo builder = No

Redis memory:
  - Base: 128MB
  - Per 100 workflow executions/hour: +64MB
  - Per 10 concurrent editors: +32MB
  - Your total: ____

=== NETWORK ===

Load balancer: Required if >1 server replica
Ingress bandwidth: ~50KB per app page load, ~5KB per query execution
WebSocket support: Required for multiplayer editing
```

---

## Common Pitfalls

### Over-provisioning Redis
Redis for ToolJet is not Redis for a cache layer. You don't need 16GB. Job payloads are small, pub/sub channels are transient. Start with 256-512MB and monitor.

### Under-provisioning PostgreSQL Connections
This is the most common production issue. The default pool size is 25 connections per ToolJet instance per database. If you're running on a managed database with a low connection limit (e.g., AWS RDS `db.t3.micro` has a limit of 87 connections), you'll hit connection exhaustion fast with just 2 server replicas + 1 worker.

**Do the math before deploying.** Use the formula above.

### Ignoring Worker Separation
Running the server and worker as the same process (single instance without `WORKER=true`) works for light usage. But when a heavy workflow executes, it competes with HTTP request handling for CPU and memory. In production, always run workers separately.

### Vertical Scaling a Node.js Process
Node.js is single-threaded. A ToolJet server instance can effectively use 1-2 CPU cores. Giving a single instance 8 CPUs is wasteful. Instead, run 4 instances with 2 CPUs each. This is why horizontal scaling matters more than vertical for ToolJet.

---

## Monitoring Checkpoints

After deploying, here's what to watch during the first week to validate your sizing:

| Metric | Healthy Range | Action if Exceeded |
|--------|--------------|-------------------|
| Server CPU | < 70% sustained | Add replica or increase CPU |
| Server Memory | < 80% of limit | Increase memory limit |
| PostgreSQL connections | < 80% of max | Check pool math, increase limit |
| PostgreSQL query latency (p95) | < 200ms | Optimize queries, add resources |
| Redis memory usage | < 70% of allocated | Increase memory |
| BullMQ queue depth | < 100 pending jobs | Add worker replicas or increase concurrency |
| Workflow execution time | < timeout threshold | Increase `WORKFLOW_TIMEOUT_SECONDS` or optimize workflows |
| API health endpoint | 200 OK, < 500ms | Investigate if failing or slow |

The health endpoint at `/api/health` is your first line of defense. Wire it into your monitoring from day one.

---

## What's Next

With your resources planned, the next chapter covers **Data & Persistence** — PostgreSQL configuration, backup strategies, migration handling, and multi-tenancy implications.
