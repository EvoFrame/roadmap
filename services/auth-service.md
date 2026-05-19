# auth-service

> **Status:** 📋 Planned (Phase 1)
> **Role:** Authentication & Authorization

Issues and validates RS256 JWTs, manages RBAC roles/permissions, OAuth2 social logins, MFA, and session lifecycle. This is the identity root of the entire platform — every other service trusts tokens it issues.

## Tech Stack

| Concern          | Technology                    |
| ---------------- | ----------------------------- |
| Framework        | FastAPI                       |
| Database         | PostgreSQL (own instance)     |
| Cache / Sessions | Redis                         |
| Token signing    | PyJWT (RS256 asymmetric)      |
| Password hashing | Argon2-cffi + bcrypt fallback |
| Scheduler        | APScheduler (token cleanup)   |

## API Endpoints

| Method | Path                           | Description                                                       |
| ------ | ------------------------------ | ----------------------------------------------------------------- |
| `POST` | `/auth/register`               | Register new user                                                 |
| `POST` | `/auth/login`                  | Issue access + refresh tokens                                     |
| `POST` | `/auth/refresh`                | Rotate refresh token                                              |
| `POST` | `/auth/logout`                 | Revoke session                                                    |
| `GET`  | `/auth/introspect`             | Token introspection (called by gateway)                           |
| `GET`  | `/auth/permissions/{user_id}`  | Resolve RBAC permissions                                          |
| `POST` | `/auth/service/token`          | Issue a short-lived service JWT (M2M)                             |
| `POST` | `/auth/service/introspect`     | Validate a service JWT (optional — services can validate locally) |
| `POST` | `/auth/verify-email`           | Verify email token                                                |
| `POST` | `/auth/password-reset/request` | Send reset link                                                   |
| `POST` | `/auth/password-reset/confirm` | Confirm + apply reset                                             |
| `POST` | `/auth/mfa/enable`             | Enable TOTP MFA                                                   |
| `POST` | `/auth/mfa/disable`            | Disable MFA                                                       |
| `POST` | `/auth/mfa/verify`             | Verify TOTP code                                                  |
| `POST` | `/auth/oauth/{provider}`       | OAuth2 social login                                               |

## Events Published (Redis Streams)

| Stream                      | Trigger                                                                                            |
| --------------------------- | -------------------------------------------------------------------------------------------------- |
| `auth.user.registered`      | New user successfully registered                                                                   |
| `auth.user.password_reset`  | Password successfully reset                                                                        |
| `auth.user.mfa_changed`     | MFA enabled or disabled                                                                            |
| `auth.service.token_issued` | Service JWT successfully issued (M2M); includes `issuer_service`, `issued_to_service`, `token_jti` |

## Security Notes

- User access tokens: RS256 JWT, 15-minute TTL
- Service tokens (M2M): RS256 JWT, 5-minute TTL — `type: "service"`, `scope: "internal"`
- Refresh tokens: opaque, stored hashed in Redis, 30-day TTL
- IP-based rate limiting on login and registration endpoints
- TOTP secrets encrypted at rest
- Password hash cost factors configurable via environment
- Only `auth-service` holds the RS256 private key — all other services validate with the public key only

## Service Client Registry

Stores the identity of every service allowed to request M2M tokens:

```
service_clients
├── id          UUID PK
├── service_id  TEXT UNIQUE   -- e.g. "api-gateway", "user-service"
├── secret_hash TEXT          -- Argon2 hash of SERVICE_SECRET
├── is_active   BOOL
├── created_at  TIMESTAMPTZ
└── updated_at  TIMESTAMPTZ
```

See [`../architecture/service-auth.md`](../architecture/service-auth.md) for the full M2M token flow.
