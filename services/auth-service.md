# auth-service

> **Status:** 🔄 Near Complete (Phase 1)
> **Role:** Authentication & Authorization

Issues and validates RS256 JWTs, manages RBAC roles/permissions, ABAC policy evaluation,
OAuth2 social logins, MFA, and session lifecycle. This is the identity root of the entire
platform — every other service trusts tokens it issues.

## Tech Stack

| Concern          | Technology                  |
| ---------------- | --------------------------- |
| Framework        | FastAPI (Python 3.13)       |
| Database         | PostgreSQL (own instance)   |
| Cache / Sessions | Redis                       |
| Token signing    | PyJWT (RS256 asymmetric)    |
| Password hashing | Argon2-cffi                 |
| OAuth2 client    | Authlib                     |
| Scheduler        | APScheduler (token cleanup) |

## API Endpoints

All routes are mounted under `/api/v1`.

### Authentication (`/users`)

| Method | Path                                   | Auth            | Description                          |
| ------ | -------------------------------------- | --------------- | ------------------------------------ |
| `POST` | `/users/register`                      | None            | Register new user                    |
| `POST` | `/users/verify-email`                  | None            | Verify email with one-time token     |
| `POST` | `/users/login`                         | None            | Issue access + refresh tokens        |
| `POST` | `/users/refresh`                       | None            | Rotate refresh token                 |
| `POST` | `/users/logout`                        | None            | Revoke refresh session               |
| `GET`  | `/users/introspect`                    | Bearer token    | Token introspection (used by gateway)|
| `GET`  | `/users/permissions/{user_id}`         | None            | Resolve user RBAC permissions        |
| `POST` | `/users/password-reset/request`        | None            | Request password reset email         |
| `POST` | `/users/password-reset/confirm`        | None            | Apply new password with reset token  |
| `POST` | `/users/oauth/{provider}`              | None            | OAuth2 social login                  |

### Account self-service (`/users/me`)

| Method   | Path             | Auth         | Description                     |
| -------- | ---------------- | ------------ | ------------------------------- |
| `GET`    | `/users/me`      | Bearer token | Get current user profile        |
| `PATCH`  | `/users/me`      | Bearer token | Update email / backup email     |
| `DELETE` | `/users/me`      | Bearer token | Soft-delete account             |

### MFA (`/users/me/mfa`)

| Method | Path                    | Auth         | Description          |
| ------ | ----------------------- | ------------ | -------------------- |
| `POST` | `/users/me/mfa/enable`  | Bearer token | Enable TOTP MFA      |
| `POST` | `/users/me/mfa/verify`  | Bearer token | Verify TOTP code     |
| `POST` | `/users/me/mfa/disable` | Bearer token | Disable TOTP MFA     |

### Admin — Users (`/users`)

Requires `X-Service-Token` + `users:read` / `users:write` permission.

| Method   | Path                 | Permission    | Description              |
| -------- | -------------------- | ------------- | ------------------------ |
| `GET`    | `/users`             | `users:read`  | List users (paginated)   |
| `GET`    | `/users/{user_id}`   | `users:read`  | Get a single user        |
| `PATCH`  | `/users/{user_id}`   | `users:write` | Update user fields       |
| `DELETE` | `/users/{user_id}`   | `users:write` | Soft-delete user         |

### Service Tokens — M2M (`/service-clients`)

| Method | Path                         | Auth              | Description                           |
| ------ | ---------------------------- | ----------------- | ------------------------------------- |
| `POST` | `/service-clients/token`     | Service secret    | Issue a short-lived service JWT (M2M) |
| `POST` | `/service-clients/introspect`| `X-Service-Token` | Validate a service JWT                |

### Admin — Service Clients (`/service-clients`)

Requires `X-Service-Token` + admin role.

| Method   | Path                            | Description                   |
| -------- | ------------------------------- | ----------------------------- |
| `POST`   | `/service-clients`              | Register a new service client |
| `GET`    | `/service-clients`              | List service clients          |
| `GET`    | `/service-clients/{service_id}` | Get a service client          |
| `PATCH`  | `/service-clients/{service_id}` | Update a service client       |
| `DELETE` | `/service-clients/{service_id}` | Deactivate a service client   |

### RBAC Management (`/rbac`)

Requires `X-Service-Token` + `rbac:read` / `rbac:write` permission.

| Method   | Path                                              | Description                      |
| -------- | ------------------------------------------------- | -------------------------------- |
| `GET`    | `/rbac/roles`                                     | List roles                       |
| `POST`   | `/rbac/roles`                                     | Create role                      |
| `GET`    | `/rbac/roles/{role_id}`                           | Get role with permissions        |
| `PATCH`  | `/rbac/roles/{role_id}`                           | Update role                      |
| `DELETE` | `/rbac/roles/{role_id}`                           | Delete role                      |
| `GET`    | `/rbac/permissions`                               | List permissions                 |
| `POST`   | `/rbac/permissions`                               | Create permission                |
| `DELETE` | `/rbac/permissions/{permission_id}`               | Delete permission                |
| `POST`   | `/rbac/roles/{role_id}/permissions`               | Assign permission to role        |
| `DELETE` | `/rbac/roles/{role_id}/permissions/{perm_id}`     | Remove permission from role      |
| `GET`    | `/rbac/users/{user_id}/roles`                     | Get user's assigned roles        |
| `POST`   | `/rbac/users/{user_id}/roles`                     | Assign role to user              |

### ABAC — Evaluation & Management (`/abac`)

Requires `X-Service-Token` + `abac:evaluate` / `abac:read` / `abac:write` permission.
See [`../architecture/service-auth.md`](../architecture/service-auth.md) for background,
and the service's own `docs/abac.md` for the full evaluation algorithm and policy model.

| Method   | Path                                           | Permission       | Description                           |
| -------- | ---------------------------------------------- | ---------------- | ------------------------------------- |
| `POST`   | `/abac/evaluate`                               | `abac:evaluate`  | Evaluate access decision (allow/deny) |
| `GET`    | `/abac/users/{user_id}/attributes`             | `abac:read`      | Get user attributes                   |
| `PUT`    | `/abac/users/{user_id}/attributes/{key}`       | `abac:write`     | Set user attribute                    |
| `DELETE` | `/abac/users/{user_id}/attributes/{key}`       | `abac:write`     | Delete user attribute                 |
| `GET`    | `/abac/policies`                               | `abac:read`      | List policies                         |
| `POST`   | `/abac/policies`                               | `abac:write`     | Create policy                         |
| `GET`    | `/abac/policies/{policy_id}`                   | `abac:read`      | Get policy with conditions            |
| `PATCH`  | `/abac/policies/{policy_id}`                   | `abac:write`     | Update policy                         |
| `DELETE` | `/abac/policies/{policy_id}`                   | `abac:write`     | Delete policy                         |
| `POST`   | `/abac/policies/{policy_id}/conditions`        | `abac:write`     | Add condition to policy               |
| `DELETE` | `/abac/conditions/{condition_id}`              | `abac:write`     | Remove condition                      |

## Events Published (Redis Streams)

| Stream                               | Trigger                                           |
| ------------------------------------ | ------------------------------------------------- |
| `auth.user.registered`               | New user registered (includes one-time `verify_token`) |
| `auth.user.password_reset_requested` | Reset token generated (includes `reset_token`)    |
| `auth.user.password_reset`           | Password successfully changed                     |
| `auth.user.mfa_changed`              | MFA enabled or disabled                           |
| `auth.user.deleted`                  | User account soft-deleted (self-service or admin) |
| `auth.service.token_issued`          | M2M JWT issued; includes `issuer_service`, `issued_to_service`, `token_jti` |
| `auth.service.client_created`        | New service client registered                     |
| `auth.service.client_deleted`        | Service client deactivated                        |

> `verify_token` and `reset_token` are included in events so downstream services (e.g.
> `notification-service`) can build and deliver the link. They are never logged.

## ABAC

`auth-service` implements a **pull-model** Attribute-Based Access Control engine.
Internal services do not enforce access decisions themselves — they POST to `/abac/evaluate`
and act on the `allow` / `deny` verdict.

The engine is **default-deny, fail-closed**: if no policy matches, the answer is always
`deny`. An explicit `deny` policy always overrides any `allow` policy.

Key concepts:

- **User attributes** — arbitrary key/value pairs attached to a user (`user_attributes` table)
- **Policies** — named rules with an `effect` (allow/deny), a `priority`, and a set of conditions
- **Conditions** — compare an attribute from `subject` / `resource` / `environment` bags
  against a literal value or another attribute using operators:
  `eq`, `neq`, `in`, `not_in`, `contains`, `gt`, `lt`, `gte`, `lte`
- **Policy scoping** — optional `scope_resource_type` and `scope_action` columns pre-filter
  policies at the DB level before the engine runs

RBAC and ABAC coexist independently. A service can use either or both. The user's RBAC
roles and permissions are automatically injected into the ABAC `subject` bag, so policies
can inspect role membership too.

## Backup Email

Users can register a secondary `backup_email` on their account (unique across all users,
with its own `backup_email_verified` flag). Managed via `PATCH /api/v1/users/me` (self) or
`PATCH /api/v1/users/{user_id}` (admin). Changing the address resets verification.

## Security Notes

- **User access tokens:** RS256 JWT, 15-minute TTL
- **Service tokens (M2M):** RS256 JWT, 5-minute TTL — `type: "service"`, `scope: "internal"`
- **Refresh tokens:** opaque composite `<session_id>:<raw_token>`, stored hashed in DB + Redis, 30-day TTL
- IP-based rate limiting on `/users/login` and `/users/register`
- TOTP secrets encrypted at rest (Fernet)
- Password hash cost factors configurable via environment
- Only `auth-service` holds the RS256 private key — all other services validate with the public key only
- `caller_service` in ABAC evaluation is always set server-side from the validated `X-Service-Token`; callers cannot spoof it
- Soft-deleted users cannot log in; their records are retained for audit purposes

## Service Client Registry

Stores the identity of every service allowed to request M2M tokens:

```text
service_clients
├── id          UUID PK
├── service_id  TEXT UNIQUE   -- e.g. "api-gateway", "user-service"
├── secret_hash TEXT          -- Argon2 hash of SERVICE_SECRET
├── is_active   BOOL
├── created_at  TIMESTAMPTZ
└── updated_at  TIMESTAMPTZ
```

See [`../architecture/service-auth.md`](../architecture/service-auth.md) for the full M2M token flow.

## Environment Variables

See [`../architecture/service-blueprint.md`](../architecture/service-blueprint.md) for the full env file strategy.

**`.env`:**

```dotenv
# ── Identity ──────────────────────────────────────────────────────────────────
SERVICE_NAME=auth-service
SERVICE_VERSION=0.1.0
SERVICE_ID=auth-service
SERVICE_SECRET=supersecret

# ── Database ──────────────────────────────────────────────────────────────────
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=auth_service
DB_PORT=5432
DATABASE_URL=postgresql+asyncpg://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}

# ── Redis ─────────────────────────────────────────────────────────────────────
REDIS_PASSWORD=redis
REDIS_PORT=6379
REDIS_URL=redis://default:${REDIS_PASSWORD}@redis:6379/0

# ── JWT / Keys ────────────────────────────────────────────────────────────────
# auth-service is the ONLY service that holds the private key
RS256_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----"
RS256_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----"
ACCESS_TOKEN_TTL=900        # 15 min — user tokens
SERVICE_TOKEN_TTL=300       # 5 min  — M2M tokens
REFRESH_TOKEN_TTL=2592000   # 30 days

# ── Password hashing ──────────────────────────────────────────────────────────
ARGON2_TIME_COST=2
ARGON2_MEMORY_COST=65536
ARGON2_PARALLELISM=2

# ── Runtime ───────────────────────────────────────────────────────────────────
APP_ENV=development
DEBUG=false
LOG_LEVEL=INFO
WORKERS=4

# ── Dev UI ────────────────────────────────────────────────────────────────────
PGADMIN_PORT=5050
PGADMIN_EMAIL=admin@local.dev
PGADMIN_PASSWORD=admin
REDIS_UI_PORT=5540

# ── Compose ───────────────────────────────────────────────────────────────────
COMPOSE_PROJECT_NAME=auth-service
```

**`.env.local`** (app runs on host, dev-stack containers mapped to localhost):

```dotenv
DATABASE_URL=postgresql+asyncpg://postgres:postgres@localhost:5432/auth_service
REDIS_URL=redis://default:redis@localhost:6379/0
```
