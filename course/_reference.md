# ToolJet Architecture Reference (for course writing)

This is a reference doc extracted from the ToolJet codebase (lts-3.16 branch) to avoid re-processing the repo for each chapter.

---

## Components

### 1. Server (Node.js / NestJS)
- Entry: `server/src/main.ts`
- Listens on `process.env.PORT` (default 3000), `LISTEN_ADDR` (default `::`)
- Serves frontend statically when `SERVE_CLIENT=true` (from `frontend/build/`)
- Can run in HTTP-only or worker mode (`WORKER=true`)
- Uses TypeORM for PostgreSQL
- Health endpoint: `/api/health`

### 2. Frontend (React)
- Built static files served by the server process
- Or can be served via nginx/CDN separately

### 3. Workers (same binary, different mode)
- Enabled by `WORKER=true` environment variable
- Processes BullMQ jobs: workflow execution, workflow scheduling, app history
- Uses Redis as job queue backend
- Concurrency: `TOOLJET_WORKFLOW_CONCURRENCY` (default 5)
- Timeout: `WORKFLOW_TIMEOUT_SECONDS` (default 60s)
- JS memory per node: `WORKFLOW_JS_MEMORY_LIMIT_MB` (default 20MB)

### 4. PostgreSQL (two databases)
- **PG_DB**: Application database (users, apps, configs) — 184 migration files
- **TOOLJET_DB**: ToolJet Database feature (user-created tables)
- Connection pool: max 25 per database instance (hardcoded in ormconfig.ts)
- Connect timeout: 5000ms
- Statement timeout (TOOLJET_DB): `TOOLJET_DB_STATEMENT_TIMEOUT` (default 60000ms)

### 5. PostgREST
- Image: `postgrest/postgrest:v12.0.2`
- Provides REST API over TOOLJET_DB
- Authenticated via `PGRST_JWT_SECRET`
- Port 80 internally

### 6. Redis
- Used for: BullMQ job queues, Pub/Sub (collaborative editing via Yjs/ioredis)
- Config: `REDIS_HOST`, `REDIS_PORT`, `REDIS_USERNAME`, `REDIS_PASSWORD`, `REDIS_DB`, `REDIS_TLS`
- Supports Redis Cluster mode
- Required for: workflows, multiplayer editing, background jobs
- NOT required for: single-instance deployments without workflows or multiplayer

---

## Resource Specs from Codebase

### Kubernetes deployment.yaml
- Replicas: 2
- Requests: 1000Mi memory, 1000m CPU
- Limits: 2000Mi memory, 2000m CPU
- Rolling update: maxUnavailable 1, maxSurge 1

### Helm values.yaml
- Requests: 1024Mi memory, 1 CPU
- Limits: 2048Mi memory, 2 CPUs
- HPA: min 1, max 1, CPU threshold 75%, memory threshold 768Mi
- PostgreSQL persistence: 8Gi
- PostgreSQL maxConnections: 1024

### Official system-requirements.md (docs)
- VM: Ubuntu 22.04+, x86 only (no arm64), 2GB RAM, 1 vCPU, 8GiB storage
- PostgreSQL: version 13.x recommended
- Redis: version 6.x recommended (for multiplayer + background jobs)

### Docker compose
- No resource limits set (relies on host resources)
- PostgreSQL image: postgres:13
- PostgREST image: postgrest/postgrest:v12.0.2

---

## Queue Configuration (server/src/modules/workflows/constants/queue-config.ts)
- Manual/Webhook priority: 0 (highest)
- Scheduled priority: 1
- Retry: 0 attempts by default, exponential backoff 2s
- Retention: 100 completed, 50 failed
- Bull Board dashboard at `/jobs` (auth: `TOOLJET_QUEUE_DASH_PASSWORD`)

---

## Key Environment Variables (resource-related)
| Variable | Default | Purpose |
|----------|---------|---------|
| PORT | 3000 | Server listen port |
| LISTEN_ADDR | :: | Server bind address |
| SERVE_CLIENT | true | Serve frontend from server |
| WORKER | false | Enable worker mode |
| REDIS_HOST | localhost | Redis host |
| REDIS_PORT | 6379 | Redis port |
| TOOLJET_WORKFLOW_CONCURRENCY | 5 | Concurrent jobs per worker |
| WORKFLOW_TIMEOUT_SECONDS | 60 | Workflow execution timeout |
| WORKFLOW_JS_MEMORY_LIMIT_MB | 20 | Memory per JS isolate in workflows |
| TOOLJET_DB_STATEMENT_TIMEOUT | 60000 | DB query timeout (ms) |
| TOOLJET_DB_BULK_UPLOAD_MAX_ROWS | 5000 | Max rows per bulk upload |
| TOOLJET_DB_BULK_UPLOAD_MAX_CSV_FILE_SIZE_MB | 5 | Max CSV file size |
| SLOW_QUERY_LOGGING_THRESHOLD | 1 | Log queries slower than N ms |

---

## Course Outline
- Module 0: Foundation (env vars done, resource planning next, then data & persistence, observability)
- Module 1: Single-Node (Docker, DigitalOcean, AWS AMI)
- Module 2: Container Orchestration (K8s concepts, Helm)
- Module 3: Managed K8s (Generic, EKS, GKE, AKS)
- Module 4: Serverless (ECS, Azure Container Apps, GCS)
- Module 5: Enterprise (OpenShift)
- Module 6: Operational Excellence (Upgrades & Maintenance)
