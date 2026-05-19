# ofelia

> **Status:** 📋 Planned (Phase 1 — shared infrastructure, bootstrapped first)
> **Role:** Docker-native cron scheduler for all platform maintenance jobs

## Description

[Ofelia](https://github.com/mcuadros/ofelia) is a lightweight job scheduler
designed for Docker environments. It reads cron schedules directly from
container labels — no separate config file, no sidecar per service. One Ofelia
instance in the global stack manages all scheduled jobs across every service.

Ofelia supports two job types used in this platform:

| Type | What it does |
|---|---|
| `job-exec` | Runs a command **inside a running container** (uses `docker exec`) |
| `job-run` | Spins up a **new container** from an image for each run, then removes it |

All audit maintenance jobs use `job-run` (the script is a standalone entrypoint,
not tied to a running process).

---

## Global stack placement

Ofelia lives in the top-level `compose.yml` alongside Redis:

```yaml
services:

  ofelia:
    image: mcuadros/ofelia:latest
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    # Ofelia reads job definitions from other containers' labels automatically.
    # No additional config needed.

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    ports:
      - "6379:6379"
```

Ofelia needs read access to the Docker socket so it can inspect labels on all
running containers and exec into them.

---

## Registering a job

Add labels to the container that should run the job. Ofelia picks them up
automatically on startup (and on container restart).

### Label schema

```
ofelia.job-run.<job-name>.schedule   = "<cron expression>"
ofelia.job-run.<job-name>.image      = "<docker image>"
ofelia.job-run.<job-name>.command    = "<command to run>"
ofelia.job-run.<job-name>.environment = "KEY=value,KEY2=value2"
ofelia.job-run.<job-name>.network    = "<docker network>"
```

Cron expressions use the standard 5-field format: `min hour dom month dow`.

---

## Registered jobs

### `audit-maintenance` — nightly at 02:00 UTC

Runs `audit_maintenance.py`: creates upcoming partitions (via pg_partman),
indexes new hot partitions, and de-indexes warm partitions.

```yaml
# In audit-service/compose.yml
audit-service:
  build: .
  labels:
    ofelia.job-run.audit-maintenance.schedule:    "0 2 * * *"
    ofelia.job-run.audit-maintenance.image:       "evoframe/audit-service:latest"
    ofelia.job-run.audit-maintenance.command:     "python audit_maintenance.py"
    ofelia.job-run.audit-maintenance.environment: "AUDIT_DB_DSN=${AUDIT_DB_DSN}"
    ofelia.job-run.audit-maintenance.network:     "evoframe-internal"
```

---

## Adding future jobs

When a new service needs a scheduled task, add `ofelia.job-run.*` labels to its
container. Ofelia will discover them on its next restart without any changes to
the global stack.

Common candidates:

| Job | Service | Suggested schedule |
|---|---|---|
| Expired refresh token cleanup | `auth-service` | `0 3 * * *` (currently handled by APScheduler in-process — migrate if load grows) |
| S3 orphan file cleanup | `file-service` | `0 4 * * *` |
| Failed notification retry sweep | `notification-service` | `*/15 * * * *` |
| Dead-letter stream inspector | any | `0 * * * *` |

---

## Security Notes

- Ofelia runs with Docker socket access — treat it as a privileged component;
  do not expose its container port externally
- Job containers run on the internal `evoframe-internal` network only
- Secrets are injected via environment variables from `.env` — never hardcoded
  in labels
- Each job uses a dedicated DB user with the minimum privileges needed
  (e.g. `audit-maintenance` user has `SELECT + CREATE INDEX + DROP INDEX` only)
