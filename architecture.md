# Art Tool Usage Tracker -- Architecture Document

> Internal system for tracking adoption and usage of marketing art tools (Maya, Photoshop, etc.).
> Provides dashboards for tool popularity, version adoption, and user churn detection.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Tech Stack](#tech-stack)
3. [Data Flow](#data-flow)
4. [Database Schema](#database-schema)
5. [API Endpoints](#api-endpoints)
6. [Go SDK](#go-sdk)
7. [Wails Desktop App](#wails-desktop-app)
8. [Project Structure](#project-structure)
9. [Deployment](#deployment)
10. [Design Decisions & Constraints](#design-decisions--constraints)
11. [Success Metrics](#success-metrics)
12. [Out of Scope](#out-of-scope)

---

## System Overview

The system spans two deployment zones:

```
+-----------------------------------------------+       +-----------------------------------------------+
|          Corporate Server (Docker)             |       |          Artist Machines (native binaries)     |
|                                                |       |                                               |
|   +-----------+       +----------------+       |       |   +------------------+    +----------------+  |
|   |  Go API   | <---> |  PostgreSQL    |       |       |   |  Art Tool        |    | Wails Desktop  |  |
|   |  Server   |       |  (persistent)  |       | <---- |   |  + Go SDK        |    | Dashboard App  |  |
|   +-----------+       +----------------+       |       |   +------------------+    +----------------+  |
|       |                                        |       |                                               |
|   [Hourly cron: refresh user_status]           |       |   SDK: fire-and-forget HTTP POST              |
+-----------------------------------------------+       |   Dashboard: HTTP GET for analytics            |
                                                         +-----------------------------------------------+
```

| Zone | Runs | Deployed As |
|------|------|-------------|
| Corporate Server | Go API + PostgreSQL | Docker containers (docker-compose) |
| Artist Machines | Art tools with embedded Go SDK, Wails dashboard app | Cross-compiled native binaries (Mac/Win/Linux) |

---

## Tech Stack

| Component | Technology | Notes |
|-----------|------------|-------|
| Backend API | Go (Gin or Chi router) | Stateless HTTP server |
| Database | PostgreSQL 16 | Dockerized, persistent volume |
| Desktop Dashboard | Wails (Go + React) | Native binary, system WebView |
| SDK | Go library | Embedded in art tool plugins |
| Server Deployment | Docker + docker-compose | Two-service stack |
| Desktop Deployment | Cross-compiled Go binaries | GOOS=darwin/windows/linux |

---

## Data Flow

Three primary data paths connect the system:

### 1. Event Ingestion (Artist -> Server)

```
Artist uses tool
  -> Go SDK fires async HTTP POST (/api/logs)
  -> 2s timeout, fails silently
  -> API server inserts row into usage_logs
  -> Server assigns timestamp (avoids clock skew)
```

### 2. Dashboard Queries (Dashboard -> Server)

```
Wails desktop app
  -> HTTP GET (/api/analytics/*)
  -> API server queries PostgreSQL
  -> Returns JSON to React frontend
```

### 3. Churn Detection (Internal Cron)

```
Hourly cron inside Go server
  -> Scans usage_logs table
  -> Refreshes user_status materialized table
  -> Flags users as active / at_risk / churned
```

---

## Database Schema

Four tables, each with a distinct role.

### `tools` -- Tool Registry

| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| name | VARCHAR(255) | UNIQUE NOT NULL |
| description | TEXT | |
| owner | VARCHAR(255) | |
| created_at | TIMESTAMPTZ | DEFAULT NOW() |

### `tool_versions` -- Version Catalog

| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| tool_id | BIGINT | FK -> tools.id |
| version | VARCHAR(100) | NOT NULL |
| released_at | TIMESTAMPTZ | |
| is_latest | BOOLEAN | DEFAULT FALSE |

- Unique constraint: `(tool_id, version)`

### `usage_logs` -- Append-Only Event Log (High Volume)

| Column | Type | Constraints |
|--------|------|-------------|
| id | BIGSERIAL | PRIMARY KEY |
| username | VARCHAR(255) | NOT NULL |
| tool_name | VARCHAR(255) | NOT NULL |
| version | VARCHAR(100) | NOT NULL |
| action | VARCHAR(255) | NOT NULL |
| timestamp | TIMESTAMPTZ | DEFAULT NOW() (server-assigned) |
| payload | JSONB | |

Indexes:

| Index | Purpose |
|-------|---------|
| `(tool_name, timestamp)` | Filter by tool over time range |
| `(username, timestamp)` | Filter by user over time range |
| `(timestamp)` | Global time-range scans |

### `user_status` -- Materialized Churn Table (Refreshed Hourly)

| Column | Type | Constraints |
|--------|------|-------------|
| username | VARCHAR(255) | PRIMARY KEY |
| last_seen | TIMESTAMPTZ | NOT NULL |
| total_events | BIGINT | DEFAULT 0 |
| status | VARCHAR(20) | CHECK IN ('active', 'at_risk', 'churned') |
| updated_at | TIMESTAMPTZ | DEFAULT NOW() |

Refreshed by an hourly cron job inside the Go server. Avoids expensive aggregation queries at read time.

---

## API Endpoints

### Ingestion

| Method | Path | Description | Response |
|--------|------|-------------|----------|
| POST | `/api/logs` | Receive a usage event from the SDK | 201 Created |

Request body:

```json
{
  "username": "jdoe",
  "tool_name": "maya",
  "version": "2024.1",
  "action": "session_start",
  "payload": { "os": "darwin", "plugin_version": "1.2.0" }
}
```

### Dashboard / Analytics

| Method | Path | Query Params | Description |
|--------|------|--------------|-------------|
| GET | `/api/logs` | `user`, `tool`, `from`, `to` | Audit table with filters |
| GET | `/api/analytics/tool-usage` | `from`, `to`, `granularity` | Usage counts grouped by tool + day |
| GET | `/api/analytics/version-adoption` | `tool` | Percentage of users on each version |
| GET | `/api/analytics/churn` | `status` | List of at-risk or churned users |
| GET | `/api/analytics/summary` | -- | KPI cards (total users, events, top tool, etc.) |

### Management

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/tools` | List all registered tools |
| GET | `/api/tools/{id}/versions` | List versions for a specific tool |

---

## Go SDK

The SDK is a lightweight Go library imported by art tool plugins. Its design principle is simple: **never block the artist's workflow**.

| Property | Detail |
|----------|--------|
| Language | Go (importable library) |
| Username | Auto-detected from OS |
| Tool identity | `tool_name` and `version` set once at initialization |
| HTTP call | Async POST to `/api/logs` |
| Timeout | 2 seconds |
| Failure mode | Silent -- errors are swallowed, never surfaced to the artist |

Usage in a plugin:

```go
import "github.com/yourorg/tool-tracker/pkg/sdk"

func main() {
    tracker := sdk.New("maya", "2024.1", "https://tracker.internal:8080")
    tracker.Track("session_start", nil)
    // non-blocking, returns immediately
}
```

---

## Wails Desktop App

The dashboard is a native desktop application built with Wails.

| Property | Detail |
|----------|--------|
| Framework | Wails (Go backend + React frontend) |
| Rendering | System WebView (not bundled Chromium) |
| Binary size | ~10-15 MB (vs ~200 MB for Electron) |
| Platforms | macOS, Windows, Linux (cross-compiled) |
| Frontend | React SPA (same codebase across all platforms) |

The React frontend communicates with the Go backend within the Wails binary, which in turn makes HTTP requests to the corporate API server.

---

## Project Structure

```
cmd/
  server/main.go                -- API server entry point
  dashboard/main.go             -- Wails desktop app entry point

internal/
  handler/
    logs.go                     -- POST/GET /api/logs
    analytics.go                -- GET /api/analytics/*
    tools.go                    -- GET /api/tools
  model/
    usage_log.go
    tool.go
    user_status.go
  store/
    postgres.go                 -- pgx connection pool setup
    logs_store.go
    analytics_store.go
    tools_store.go
  cron/
    refresh_status.go           -- hourly user_status refresh job

pkg/
  sdk/
    tracker.go                  -- Go SDK client library

frontend/
  src/                          -- React SPA (shared by Wails desktop app)

Dockerfile
docker-compose.yml
```

Key conventions:

- `cmd/` -- Application entry points (one per binary)
- `internal/` -- Private server code (handlers, models, stores, cron)
- `pkg/` -- Public libraries (SDK, consumed by external art tool plugins)
- `frontend/` -- React SPA, bundled into the Wails binary at build time

---

## Deployment

### Server (Docker)

docker-compose runs two services:

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: tracker
      POSTGRES_USER: tracker
      POSTGRES_PASSWORD: ${DB_PASSWORD}

  api:
    build: .
    depends_on:
      - db
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://tracker:${DB_PASSWORD}@db:5432/tracker?sslmode=disable
```

### Desktop (Cross-Compiled Binaries)

Built from a single machine, targeting all three platforms:

```bash
# macOS
GOOS=darwin  GOARCH=amd64 go build -o tracker-dashboard-mac     ./cmd/dashboard

# Windows
GOOS=windows GOARCH=amd64 go build -o tracker-dashboard-win.exe ./cmd/dashboard

# Linux
GOOS=linux   GOARCH=amd64 go build -o tracker-dashboard-linux   ./cmd/dashboard
```

Distributed to artist machines via internal file share or package manager.

---

## Scaling: Multiple Instances & Load Balancing

The Go API server is **stateless** — it holds no session data, no in-memory cache, and no local state. This means you can run multiple instances behind a load balancer with zero code changes.

### Why Multiple Instances?

| Concern | Single Instance | Multiple Instances |
|---------|----------------|-------------------|
| Availability | One crash = full outage | Other instances keep serving |
| Ingestion throughput | Bottlenecked by one process | Scales horizontally with tool adoption |
| Zero-downtime deploys | Must restart, logs are lost during restart | Rolling deploy — at least one instance always up |

### Architecture with Load Balancer

```
                        ┌─────────────────┐
                        │  Load Balancer   │
                        │  (Nginx/HAProxy) │
                        └────────┬────────┘
                     ┌───────────┼───────────┐
                     ▼           ▼           ▼
              ┌──────────┐ ┌──────────┐ ┌──────────┐
              │  Go API  │ │  Go API  │ │  Go API  │
              │  :8080   │ │  :8081   │ │  :8082   │
              └────┬─────┘ └────┬─────┘ └────┬─────┘
                   └────────────┼────────────┘
                                ▼
                        ┌──────────────┐
                        │  PostgreSQL   │
                        └──────────────┘
```

### docker-compose with scaling

```yaml
services:
  lb:
    image: nginx:alpine
    ports:
      - "8080:80"
    depends_on:
      - api
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro

  api:
    build: .
    deploy:
      replicas: 3
    environment:
      DATABASE_URL: postgres://tracker:${DB_PASSWORD}@db:5432/tracker?sslmode=disable

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
```

Scale up on demand:

```bash
docker-compose up --scale api=5
```

### Cron Job Consideration

With multiple instances, the hourly `user_status` refresh cron would run in **every instance**. To avoid duplicate work:

- **Option A: Leader election** — use PostgreSQL advisory locks. Only the instance that acquires the lock runs the cron:
  ```sql
  SELECT pg_try_advisory_lock(1);  -- returns true for one instance only
  ```
- **Option B: External scheduler** — run the cron as a separate one-off container (`docker-compose run --rm api refresh-status`) triggered by the host's crontab.

**Recommendation:** Option A (advisory locks) — keeps everything self-contained, no external scheduler needed.

### Load Balancing Strategy

| Strategy | Use For |
|----------|---------|
| **Round-robin** | `POST /api/logs` — all instances are equal, distribute evenly |
| **Round-robin** | `GET /api/analytics/*` — stateless reads, any instance works |
| **Health checks** | Nginx checks `/health` on each instance, removes unhealthy ones |

No sticky sessions needed — every request is independent.

---

## Design Decisions & Constraints

| Decision | Rationale |
|----------|-----------|
| Server-assigned timestamps | Avoids clock skew across artist machines |
| Denormalized `tool_name` in `usage_logs` | Eliminates FK lookup on every POST, keeps ingestion fast |
| `user_status` as precomputed table | Churn queries would be expensive over the full `usage_logs` table; hourly refresh is sufficient |
| SDK fails silently | Artist workflow must never be blocked by telemetry |
| No authentication | IP-restricted corporate network; auth is out of scope |
| PostgreSQL JSONB for `payload` | Flexible schema for tool-specific metadata; indexable when needed |
| Wails over Electron | 10x smaller binary (~15 MB vs ~200 MB), uses system WebView |
| Append-only `usage_logs` | No updates or deletes; simplifies concurrency and enables future partitioning |

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Tool adoption | 80% of artists onboarded within 30 days |
| Churn detection | 14 days of inactivity flags a user as "at risk" |
| Engagement improvement | 15% increase driven by data-informed tool improvements |

---

## Out of Scope

The following are explicitly excluded from the current phase:

- Cloud / AWS migration
- Authentication and role-based access control (RBAC)
- Automated email or Slack reporting
