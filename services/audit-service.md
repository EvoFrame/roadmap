# audit-service

> **Status:** 📋 Planned (Phase 8)  
> **Role:** Immutable security and compliance event log

## Description

Consumes every event from every Redis Stream and writes a tamper-evident, append-only audit record. Provides a searchable read-only API for administrators. Records are **never updated or deleted** — enforced at both the application and database level.

## Tech Stack

| Concern | Technology |
|---|---|
| Framework | FastAPI |
| Database | PostgreSQL (own instance, append-only, range-partitioned) |
| Partition management | `pg_partman` extension |
| Event consumption | Redis Streams (consumer group `audit-service`) |

## API Endpoints

All endpoints require **admin role**.

| Method | Path | Description |
|---|---|---|
| `GET` | `/audit/events` | Paginated, filterable audit log |
| `GET` | `/audit/events/{id}` | Single audit event detail |

### Query filters for `GET /audit/events`

| Parameter | Type | Description |
|---|---|---|
| `user_id` | UUID | Filter by actor |
| `event_type` | string | Filter by stream key (e.g. `auth.user.registered`) |
| `from` | ISO 8601 | Start of time range |
| `to` | ISO 8601 | End of time range |
| `page` / `page_size` | int | Pagination |

## Events Consumed

**All streams** — every event published on the Redis event bus is consumed and recorded.

| Stream | Retention class |
|---|---|
| `auth.access.denied` | Security |
| `auth.service.token_issued` | Security |
| `auth.user.registered` | Identity / billing |
| `auth.user.password_reset` | Identity / billing |
| `auth.user.mfa_changed` | Identity / billing |
| `user.profile.updated` | Business activity |
| `user.account.deleted` | Identity / billing |
| `team.member.invited` | Business activity |
| `team.member.joined` | Business activity |
| `team.member.removed` | Business activity |
| `team.deleted` | Business activity |
| `billing.subscription.activated` | Identity / billing |
| `billing.subscription.cancelled` | Identity / billing |
| `billing.payment.failed` | Identity / billing |
| `file.uploaded` | Business activity |
| `file.deleted` | Business activity |

## Events Published

None.

## Data Model

### Why three tables, not one

A single `audit_events` table partitioned by time means every monthly partition
holds a **mix** of all retention classes. `pg_partman` has one retention window
per table — setting it to 2 years (the longest) means 90-day business events
are kept for 2 years; setting it to 90 days destroys security events. The
`retention_class` column cannot help because `pg_partman` drops the entire
partition, not individual rows.

**Solution: one table per retention class.** Each table gets its own
`pg_partman` configuration. The retention window is enforced correctly by
PostgreSQL itself — no cron jobs, no row-level deletes.

```
Redis Stream consumer
        │
        ├─▶ audit_events_security   (pg_partman: 2 years)
        ├─▶ audit_events_identity   (pg_partman: 1 year)
        └─▶ audit_events_business   (pg_partman: 90 days)

Admin API
        └─▶ audit_events_view  (UNION ALL of the three tables)
```

### Shared column schema

All three tables have the same columns:

```sql
-- Replace <table> with: audit_events_security | audit_events_identity | audit_events_business
CREATE TABLE <table> (
    id                UUID         NOT NULL DEFAULT gen_random_uuid(),
    stream            TEXT         NOT NULL,  -- stream key, e.g. "auth.access.denied"
    source_service    TEXT         NOT NULL,  -- service that published the event
    issuer_service    TEXT,                   -- NULL for non-s2s events
    issued_to_service TEXT,                   -- NULL for non-s2s events
    payload           JSONB        NOT NULL,  -- full flat event payload (no secrets)
    recorded_at       TIMESTAMPTZ  NOT NULL DEFAULT now(),
    PRIMARY KEY (id, recorded_at)             -- partition key must be in PK
)
PARTITION BY RANGE (recorded_at);
```

### Stream → table routing

The consumer derives the target table from the stream name at write time:

```python
SECURITY_STREAMS = {"auth.access.denied", "auth.service.token_issued"}
IDENTITY_STREAMS = {
    "auth.user.registered", "auth.user.password_reset", "auth.user.mfa_changed",
    "user.account.deleted",
    "billing.subscription.activated", "billing.subscription.cancelled",
    "billing.payment.failed",
}

def table_for(stream: str) -> str:
    if stream in SECURITY_STREAMS:
        return "audit_events_security"
    if stream in IDENTITY_STREAMS:
        return "audit_events_identity"
    return "audit_events_business"
```

### pg_partman — one config per table

```sql
-- security: 2-year retention
SELECT partman.create_parent('public.audit_events_security', 'recorded_at', '1 month', p_premake => 3);
UPDATE partman.part_config SET retention = '2 years', retention_keep_table = false
  WHERE parent_table = 'public.audit_events_security';

-- identity / billing: 1-year retention
SELECT partman.create_parent('public.audit_events_identity', 'recorded_at', '1 month', p_premake => 3);
UPDATE partman.part_config SET retention = '1 year', retention_keep_table = false
  WHERE parent_table = 'public.audit_events_identity';

-- business activity: 90-day retention
SELECT partman.create_parent('public.audit_events_business', 'recorded_at', '1 month', p_premake => 3);
UPDATE partman.part_config SET retention = '90 days', retention_keep_table = false
  WHERE parent_table = 'public.audit_events_business';
```

`pg_partman`'s background worker runs nightly. Dropping an expired partition is
instant and lock-free — no `DELETE` statements involved.

### Unified read view

```sql
CREATE VIEW audit_events_view AS
    SELECT *, 'security' AS retention_class FROM audit_events_security
    UNION ALL
    SELECT *, 'identity' AS retention_class FROM audit_events_identity
    UNION ALL
    SELECT *, 'business' AS retention_class FROM audit_events_business;
```

The API queries `audit_events_view`. When filtering by `stream` or `recorded_at`
range, the query planner pushes the predicate through the UNION and prunes
irrelevant partitions in each child table automatically.

### Indexes — tiered by partition age

```sql
-- Applied to every new partition (all three tables):
CREATE INDEX ON <table>_YYYY_MM (stream, recorded_at DESC);
CREATE INDEX ON <table>_YYYY_MM (source_service, recorded_at DESC);
CREATE INDEX ON <table>_YYYY_MM USING GIN (payload);

-- GIN index dropped on partitions older than 90 days
-- (warm/cold data is rarely queried by payload content):
DROP INDEX <table>_YYYY_MM_payload_idx;
```

---

## Retention policy

| Table | Streams | Retention |
|---|---|---|
| `audit_events_security` | `auth.access.denied`, `auth.service.token_issued` | **2 years** |
| `audit_events_identity` | `auth.user.*`, `user.account.deleted`, `billing.*` | **1 year** |
| `audit_events_business` | all other streams | **90 days** |

**Retention tiers within each table:**

```
0 → 90 days   HOT    live partition  full indexes (stream+time, source+time, GIN)
90d → max     WARM   live partition  stream+time + source+time only  (GIN dropped)
past max      COLD   pg_partman drops partition automatically
```

---

**Immutability enforcement:**

- DB user has `INSERT` privilege only — no `UPDATE`, `DELETE`, `TRUNCATE`
- No soft-delete or update endpoint exists in the service
- PostgreSQL row-level security (RLS) blocks any `UPDATE`/`DELETE` even from the service role

---

## Maintenance script

Run nightly (Docker cron or Kubernetes CronJob). Does three things in order:

1. **pg_partman maintenance** — creates upcoming monthly partitions, drops expired ones
2. **Index new hot partitions** — any partition `< 90 days` old gets the full index set if missing
3. **De-index warm partitions** — any partition `>= 90 days` old has its GIN index dropped

```python
#!/usr/bin/env python3
"""
audit_maintenance.py — nightly routine for audit_events_* partitioned tables.

Usage:
    AUDIT_DB_DSN=postgresql://user:pass@host/db python audit_maintenance.py

Run via Docker cron or Kubernetes CronJob once per night.
"""
import asyncio
import logging
import os
from datetime import datetime, timezone, timedelta

import asyncpg

DSN        = os.environ["AUDIT_DB_DSN"]
HOT_WINDOW = timedelta(days=90)

PARENT_TABLES = [
    "audit_events_security",
    "audit_events_identity",
    "audit_events_business",
]

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger(__name__)


# ---------------------------------------------------------------------------
# 1. pg_partman — create future partitions + drop expired ones
# ---------------------------------------------------------------------------

async def run_partman(conn: asyncpg.Connection) -> None:
    log.info("Running pg_partman maintenance...")
    await conn.execute("SELECT partman.run_maintenance_proc()")
    log.info("pg_partman maintenance done.")


# ---------------------------------------------------------------------------
# 2 & 3. Index management
# ---------------------------------------------------------------------------

def _partition_start(bound_expr: str) -> datetime | None:
    """Parse lower bound from: FOR VALUES FROM ('2026-01-01 00:00:00+00') TO (...)"""
    try:
        raw = bound_expr.split("FROM ('")[1].split("'")[0]
        return datetime.fromisoformat(raw).replace(tzinfo=timezone.utc)
    except Exception:
        return None


async def _existing_indexes(conn: asyncpg.Connection, partition: str) -> set[str]:
    rows = await conn.fetch(
        "SELECT indexname FROM pg_indexes WHERE tablename = $1", partition
    )
    return {r["indexname"] for r in rows}


async def _ensure_hot_indexes(conn: asyncpg.Connection, partition: str) -> None:
    """Create stream+time, source+time, and GIN indexes if missing."""
    existing = await _existing_indexes(conn, partition)

    indexes = {
        f"{partition}_stream_idx":  f"CREATE INDEX IF NOT EXISTS {partition}_stream_idx"
                                    f" ON {partition} (stream, recorded_at DESC)",
        f"{partition}_source_idx":  f"CREATE INDEX IF NOT EXISTS {partition}_source_idx"
                                    f" ON {partition} (source_service, recorded_at DESC)",
        f"{partition}_gin_idx":     f"CREATE INDEX IF NOT EXISTS {partition}_gin_idx"
                                    f" ON {partition} USING GIN (payload)",
    }

    for name, ddl in indexes.items():
        if name not in existing:
            log.info("  [CREATE] %s", name)
            await conn.execute(ddl)


async def _drop_gin_index(conn: asyncpg.Connection, partition: str) -> None:
    """Drop the GIN payload index from a warm partition if it still exists."""
    gin_idx = f"{partition}_gin_idx"
    exists = await conn.fetchval(
        "SELECT indexname FROM pg_indexes WHERE tablename = $1 AND indexname = $2",
        partition, gin_idx,
    )
    if exists:
        log.info("  [DROP GIN] %s", gin_idx)
        await conn.execute(f"DROP INDEX IF EXISTS {gin_idx}")


async def maintain_indexes(conn: asyncpg.Connection) -> None:
    now = datetime.now(timezone.utc)

    for parent in PARENT_TABLES:
        partitions = await conn.fetch(
            """
            SELECT c.relname AS name, pg_get_expr(c.relpartbound, c.oid) AS bound
            FROM pg_inherits i
            JOIN pg_class c ON c.oid = i.inhrelid
            JOIN pg_class p ON p.oid = i.inhparentid
            WHERE p.relname = $1
            ORDER BY c.relname
            """,
            parent,
        )
        log.info("Table %s — %d partitions", parent, len(partitions))

        for row in partitions:
            name  = row["name"]
            start = _partition_start(row["bound"])

            if start is None:
                log.warning("  Could not parse bound for %s, skipping", name)
                continue

            age = now - start

            if age < HOT_WINDOW:
                log.info("  [HOT  ] %s (%d days old)", name, age.days)
                await _ensure_hot_indexes(conn, name)
            else:
                log.info("  [WARM ] %s (%d days old)", name, age.days)
                await _drop_gin_index(conn, name)


# ---------------------------------------------------------------------------
# Entrypoint
# ---------------------------------------------------------------------------

async def main() -> None:
    log.info("=== Audit maintenance starting ===")
    conn = await asyncpg.connect(DSN)
    try:
        await run_partman(conn)
        await maintain_indexes(conn)
    finally:
        await conn.close()
    log.info("=== Audit maintenance complete ===")


if __name__ == "__main__":
    asyncio.run(main())
```

### Docker Compose cron service

```yaml
audit-maintenance:
  build: ./audit-service
  command: ["python", "audit_maintenance.py"]
  environment:
    AUDIT_DB_DSN: ${AUDIT_DB_DSN}
  restart: "no"
  # Run nightly at 02:00 UTC via a separate scheduler (e.g. ofelia or supercronic)
  labels:
    ofelia.enabled: "true"
    ofelia.job-run.audit-maintenance.schedule: "0 2 * * *"
    ofelia.job-run.audit-maintenance.container: "audit-maintenance"
```

## Security Notes

- Read API is admin-only (enforced via `X-User-Roles` header)
- Consumer group uses Redis Streams ACK — events committed only after successful write to PostgreSQL
- Payload JSONB is stored as-is — redact secrets upstream, before publishing events
- `issuer_service` / `issued_to_service` stored as plain text for query performance (no FK — audit table must be self-contained)
- Maintenance script runs with a separate DB user that has `SELECT` + `CREATE INDEX` + `DROP INDEX` only — no `INSERT`/`DELETE`
