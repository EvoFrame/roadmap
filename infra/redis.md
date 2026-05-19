# Redis — Shared Infrastructure

> **Image:** `redis:7-alpine`
> **Role:** Event bus · service token cache · user session store

---

## Responsibilities in EvoFrame

| Use case | Mechanism | Who uses it |
|---|---|---|
| Async event bus | Redis Streams (`XADD` / `XREADGROUP`) | All services |
| Service token cache | `SET NX EX` + distributed lock | All services (via `ServiceTokenCache`) |
| User session / refresh tokens | `SET EX` keyed by token hash | `auth-service` |
| Dead-letter storage | Stream key `dead-letter:<stream>:<group>` | All consumers |

Full event bus documentation: [architecture/event-bus.md](../architecture/event-bus.md)
Full service token cache documentation: [architecture/service-auth.md](../architecture/service-auth.md)

---

## Configuration

```bash
# Shared across all services
REDIS_URL=redis://redis:6379/0
```

One Redis instance is shared by all services — database `0` for all use cases.
Namespace collisions are avoided by key prefixes:

| Prefix | Owner | Example key |
|---|---|---|
| *(stream name)* | event bus | `auth.user.registered` |
| `svc_token:<service_id>` | service token cache | `svc_token:user-service` |
| `svc_token_lock:<service_id>` | distributed lock | `svc_token_lock:user-service` |
| `refresh:<token_hash>` | auth-service sessions | `refresh:a1b2c3...` |
| `dead-letter:<stream>:<group>` | dead-letter store | `dead-letter:auth.user.registered:notification-service` |

---

## Docker Compose (shared stack)

```yaml
redis:
  image: redis:7-alpine
  restart: unless-stopped
  volumes:
    - redis_data:/data
  command: redis-server --appendonly yes   # AOF persistence
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 5s
    timeout: 3s
    retries: 5
  networks:
    - evoframe-internal

volumes:
  redis_data:
```

AOF persistence (`--appendonly yes`) ensures the event bus and session data
survive container restarts without data loss.

---

## Operational notes

- **Memory pressure:** each stream is capped with `XADD ... MAXLEN ~50000`.
  Monitor stream lengths via `XLEN <stream>` or Prometheus.
- **Eviction policy:** set `maxmemory-policy noeviction` so Redis rejects writes
  rather than silently dropping keys when memory is full.
- **Monitoring:** expose Redis metrics to Prometheus via
  [`redis_exporter`](https://github.com/oliver006/redis_exporter).
