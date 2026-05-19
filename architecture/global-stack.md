# Global Stack

The global compose stack orchestrates all EvoFrame services and shared
infrastructure from a single root `compose.yml` at the repo root.

---

## Overview

| File | Purpose |
|---|---|
| `compose.yml` | Root — `include:` directives only |
| `.docker/compose/redis.compose.yml` | Shared Redis event bus + cache |
| `.docker/compose/ofelia.compose.yml` | Cron scheduler |
| `.docker/compose/monitoring.compose.yml` | Prometheus + Grafana |
| `.env.example` | All env vars — copy to `.env` |
| `taskfile.yml` | Global task runner |
| `taskfiles/infra.yml` | Infra-specific tasks |

---

## Network topology

```text
evoframe-internal       bridge — all services + infra communicate here
evoframe_redis          Redis private network (Redis + UI + exporter)
evoframe_monitoring     Monitoring private network (Prometheus + Grafana)
evoframe_<svc>_db       Per-service DB private network (PostgreSQL + exporter)
```

`evoframe-internal` is the only cross-cutting network. Services must not
expose ports to `0.0.0.0` — all host-bound ports use `127.0.0.1:`.

The network is pre-created by `task network` (or `task infra:network`).
Per-service standalone composes reference it as a named network; Docker
reuses it if it already exists.

---

## Compose profiles

| Profile | What starts |
|---|---|
| `services` | Core containers (default, always needed) |
| `ui` | Admin UIs — RedisInsight, pgAdmin per service (opt-in) |
| `monitoring` | Exporters + Prometheus + Grafana (opt-in) |

```sh
# Core infra only
docker compose --profile services up -d

# Core + all admin UIs
docker compose --profile services --profile ui up -d

# Core + monitoring
docker compose --profile services --profile monitoring up -d
```

Or use the task aliases:

```sh
task up              # services profile
task up-ui           # services + ui
task up-monitoring   # services + monitoring
task up-all          # all profiles
```

---

## Adding a service to the global stack

When a new service is scaffolded, uncomment its lines in the root
`compose.yml`:

```yaml
include:
  # ...existing infra...

  - path: user-service/.docker/compose/app.compose.yml
  - path: user-service/.docker/compose/db.compose.yml
```

The `app.compose.yml` for each service references `evoframe-internal` as an
external network — services discover each other by container name.

---

## First-time setup

```sh
# 1. Copy and fill in credentials
cp .env.example .env

# 2. Enter devbox shell
devbox shell

# 3. Create the shared network
task network

# 4. Start shared infra
task up

# 5. (Optional) start monitoring
task up-monitoring
```

---

## Environment variables

All credentials live in the root `.env` file (gitignored). See `.env.example`
for the full list with inline documentation.

Port allocation:

| Service | App port | pgAdmin port |
|---|---|---|
| api-gateway | 8000 | — |
| auth-service | 8001 | 5051 |
| user-service | 8002 | 5052 |
| team-service | 8003 | 5053 |
| billing-service | 8004 | 5054 |
| notification-service | 8005 | 5055 |
| file-service | 8006 | 5056 |
| audit-service | 8007 | 5057 |
| Redis UI | — | 5540 |
| Prometheus | — | 9090 |
| Grafana | — | 3000 |

---

## Monitoring

Prometheus scrapes `/metrics` from every service container on the
`evoframe-internal` network. The scrape config lives at
`.docker/monitoring/prometheus.yml`.

Grafana is pre-provisioned with Prometheus as its default datasource
(`.docker/monitoring/grafana/provisioning/datasources/prometheus.yml`).

---

## Standalone vs global dev

| Mode | How to run | Redis used | Network |
|---|---|---|---|
| Standalone (inside service dir) | `task up` | Per-service Redis | Each service creates `evoframe-internal` if absent |
| Global stack (repo root) | `task up` | Shared `evoframe-redis` | Global stack owns `evoframe-internal` |

When switching from standalone to global, set `REDIS_HOST=evoframe-redis`
in the relevant service `.env`.
