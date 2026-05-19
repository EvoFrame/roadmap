# PostgreSQL — Shared Infrastructure

> **Image:** `postgres:16-alpine`
> **Role:** Primary relational store — one isolated instance per service

---

## Isolation model

Every business service runs its own PostgreSQL container with its own database,
credentials, and volume. Services never share a database — there are no
cross-service foreign keys and no cross-service queries.

```
auth-service     → auth_db    (users · roles · service_clients · refresh tokens)
user-service     → user_db    (profiles · preferences)
team-service     → team_db    (teams · memberships · invites)
billing-service  → billing_db (subscriptions · invoices · Stripe events)
notification-service → notif_db (notification log · preferences)
file-service     → file_db    (file metadata · access rules)
audit-service    → audit_db   (audit_events_security · audit_events_identity · audit_events_business)
```

---

## Per-service Docker Compose snippet

Each service's `compose.yml` includes its own `db` service:

```yaml
db:
  image: postgres:16-alpine
  environment:
    POSTGRES_USER:     ${DB_USER}
    POSTGRES_PASSWORD: ${DB_PASSWORD}
    POSTGRES_DB:       ${DB_NAME}
  volumes:
    - db_data:/var/lib/postgresql/data
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
    interval: 5s
    timeout: 3s
    retries: 5
  networks:
    - evoframe-internal

volumes:
  db_data:
```

---

## Migrations

All services use **Alembic** for schema migrations, wired to **SQLModel** models.

```bash
# Generate a new migration
task migrate-new name=add_avatar_column

# Apply all pending migrations
task migrate
```

Migrations run automatically on service startup via `entrypoint.sh`:

```sh
#!/bin/sh
set -e

alembic -c resources/migrations/alembic.ini upgrade head

if [ "${APP_ENV:-production}" = "development" ]; then
    exec uvicorn resources.server:app --reload --host 0.0.0.0 --port 8080
else
    exec gunicorn resources.server:app \
        -k uvicorn.workers.UvicornWorker \
        -w "${WORKERS:-4}" \
        --bind 0.0.0.0:8080 \
        --forwarded-allow-ips="*" \
        --access-logfile - \
        --error-logfile -
fi
```

---

## DB user privileges

Each service connects with a dedicated DB user that has the minimum required
privileges:

| Service | Privileges |
|---|---|
| All business services | `SELECT · INSERT · UPDATE · DELETE` on own tables |
| `audit-service` (writer) | `INSERT` only on `audit_events_*` tables |
| `audit-service` (maintenance) | `SELECT · CREATE INDEX · DROP INDEX` only |

The `SUPERUSER` role is never used by application code.

---

## Extensions

| Extension | Used by | Purpose |
|---|---|---|
| `pgcrypto` | All services | `gen_random_uuid()` for UUID primary keys |
| `pg_partman` | `audit-service` | Monthly partition management with auto-drop |

```sql
-- Run once on DB init:
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS pg_partman;
```

---

## Operational notes

- **Backups:** use `pg_dump` scheduled via Ofelia or a managed backup tool in production.
- **Connection pooling:** use [PgBouncer](https://www.pgbouncer.org/) in front of
  each instance in production to limit connection overhead from multiple service replicas.
- **Monitoring:** expose metrics to Prometheus via
  [`postgres_exporter`](https://github.com/prometheus-community/postgres_exporter).
