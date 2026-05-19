# EvoFrame — Wiki

> Platform plan and architecture reference for the EvoFrame microservices stack.

---

## 📌 Architecture decisions

| Concern              | Decision                                                                   |
| -------------------- | -------------------------------------------------------------------------- |
| Language / framework | Python 3.13 + FastAPI (all services)                                       |
| External entry point | Dedicated `api-gateway` (reverse proxy + auth enforcement)                 |
| Sync communication   | REST over internal Docker network                                          |
| Async communication  | Redis Streams (consumer groups per subscriber)                             |
| Databases            | One isolated PostgreSQL instance **per service**                           |
| File storage         | Local filesystem (dev) / S3-compatible MinIO or AWS S3 (prod)              |
| Payments             | Stripe                                                                     |
| Auth tokens          | RS256 JWT (asymmetric) — issued by `auth-service`                          |
| Observability        | Prometheus + Grafana + Promtail + Loki (shared stack)                      |
| Scheduled jobs       | [Ofelia](infra/ofelia.md) — Docker-native cron scheduler (label-driven) |

---

## 🗂 Wiki pages

### Architecture

| Page                                                      | Description                                                                                   |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| [overview.md](architecture/overview.md)                   | Service topology, auth flow, event bus, and security layer diagrams                           |
| [service-blueprint.md](architecture/service-blueprint.md) | Full internal structure every service follows — folder layout, patterns, templates, checklist |
| [service-auth.md](architecture/service-auth.md)           | Service-to-service JWT auth — token issuance, caching, validation middleware                  |
| [event-bus.md](architecture/event-bus.md)                 | Redis Streams event bus — publishing, consuming, retry, dead-letter, full event catalogue     |
| [global-stack.md](architecture/global-stack.md)           | Global compose stack — network topology, profiles, first-time setup, port map                 |

### Services

| Service                                                  | Phase | Role                                                          |
| -------------------------------------------------------- | ----- | ------------------------------------------------------------- |
| [auth-service](services/auth-service.md)                 | 1     | JWT issuance · RBAC · MFA · OAuth2 · M2M tokens               |
| [api-gateway](services/api-gateway.md)                   | 2     | Single entry point · JWT validation · routing · rate limiting |
| [user-service](services/user-service.md)                 | 3     | User profiles & preferences                                   |
| [team-service](services/team-service.md)                 | 4     | Teams · memberships · invitations                             |
| [billing-service](services/billing-service.md)           | 5     | Stripe subscriptions & invoices                               |
| [notification-service](services/notification-service.md) | 6     | Email + in-app notifications                                  |
| [file-service](services/file-service.md)                 | 7     | File upload · storage · secure download                       |
| [audit-service](services/audit-service.md)               | 8     | Immutable security & compliance log                           |

### Infrastructure

| Component | Spec | Role |
|---|---|---|
| Ofelia | [infra/ofelia.md](infra/ofelia.md) | Docker-native cron scheduler — nightly maintenance jobs |
| Redis | [infra/redis.md](infra/redis.md) | Event bus (Streams) · service token cache · session store |
| PostgreSQL | [infra/postgres.md](infra/postgres.md) | Isolated DB per service · Alembic migrations · pg_partman |
| Prometheus + Grafana | [architecture/global-stack.md](architecture/global-stack.md) | Metrics & dashboards |
| Promtail + Loki | shared stack | Log aggregation |

---

## 🏗 Build order

```
Phase 0 ── shared infra          (Redis · Ofelia · observability stack)
Phase 1 ── auth-service          (JWT · RBAC · MFA · OAuth2 · M2M tokens)
Phase 2 ── api-gateway           (routing · JWT introspection · rate limiting)
Phase 3 ── user-service          (profiles) + Redis Streams bus setup
Phase 4 ── team-service          (teams · memberships · invites)
Phase 5 ── billing-service       (Stripe subscriptions + webhooks)
Phase 6 ── notification-service  (email + in-app inbox)
Phase 7 ── file-service          (upload · presigned URLs · MinIO stack)
Phase 8 ── audit-service         (append-only compliance log)
Phase 9 ── Security hardening    (mTLS · load test · penetration review)
```

---

## 🔒 Security layers

1. TLS termination at `api-gateway` (nginx/Caddy in production)
2. RS256 JWT validation on every inbound request at the gateway
3. Short-lived service JWTs for service-to-service calls (see [service-auth.md](architecture/service-auth.md))
4. RBAC enforced per service (`X-User-Roles` header injected by gateway)
5. Isolated PostgreSQL per service — minimum DB user privileges
6. Append-only audit log — `INSERT`-only DB user + RLS
7. Redis Streams — consumer groups with dead-letter handling
8. Stripe webhooks verified via HMAC signature
9. Presigned file URLs with short TTL
10. Input validation via Pydantic on all services
