# api-gateway

> **Status:** 📋 Planned (Phase 2)  
> **Role:** Single external entry point — routing, JWT enforcement, rate limiting

## Description

Reverse-proxies all inbound HTTPS requests to the correct downstream service. Validates JWTs by calling `auth-service /auth/introspect` on every request, then injects trusted headers (`X-User-Id`, `X-User-Roles`, `X-Internal-Token`) before forwarding. Contains **no business logic** — purely infrastructure.

## Tech Stack

| Concern | Technology |
|---|---|
| Framework | FastAPI |
| HTTP proxy | `httpx` (async) |
| Rate limiting | Redis (sliding window counters) |
| Database | None |

## Routing Table

| Public prefix | Upstream service | Notes |
|---|---|---|
| `/auth/*` | `auth-service` | Pass-through, no JWT required on login/register |
| `/users/*` | `user-service` | JWT required |
| `/teams/*` | `team-service` | JWT required |
| `/billing/*` | `billing-service` | JWT required |
| `/notifications/*` | `notification-service` | JWT required |
| `/files/*` | `file-service` | JWT required |
| `/audit/*` | `audit-service` | JWT required, admin role required |

## Request Processing Pipeline

```
Client request
    │
    ▼
① CORS check (reject unknown origins)
    │
    ▼
② Rate limit check (per-IP sliding window, Redis)
    │
    ▼
③ Route match → determine upstream
    │
    ▼
④ JWT present? → introspect via auth-service
    │             inject X-User-Id, X-User-Roles, X-Team-Id
    ▼
⑤ Inject X-Internal-Token header
    │
    ▼
⑥ Proxy request to upstream
    │
    ▼
⑦ Strip internal headers from response
    │
    ▼
Client response
```

## Security Notes

- TLS termination here (nginx/Caddy in production, passthrough in dev)
- `X-User-Id`, `X-User-Roles`, `X-Internal-Token` are **stripped from incoming requests** before processing — they can only be set by the gateway
- Rate limits: global 1000 req/min per IP, 200 req/min per user
- CORS: strict allowlist, no wildcard in production
- Request/response logging strips Authorization headers and PII
- Health check endpoint (`GET /health`) bypasses auth

## Events Published

| Stream | Trigger | Payload |
|---|---|---|
| `auth.access.denied` | Any request rejected due to invalid, expired, or missing user JWT | `ip`, `endpoint`, `method`, `reason`, `timestamp` |

### `auth.access.denied` payload

```python
await redis.xadd("auth.access.denied", {
    "ip":        request.client.host,
    "endpoint":  str(request.url.path),
    "method":    request.method,
    "reason":    "expired" | "invalid_signature" | "missing" | "wrong_type",
    "timestamp": datetime.utcnow().isoformat(),
    # user_id omitted — token may not be decodable
})
```

**Note:** service-to-service token failures are **not** published here — they go
to structured logs + Prometheus metrics only (caller identity is unverifiable on
a bad token). See [`architecture/service-auth.md`](../architecture/service-auth.md).
