# Service Blueprint

Every EvoFrame service is scaffolded from this blueprint. Deviations must be
documented in the service's own spec file.

---

## Folder structure

```
<service-name>/
├── .docker/
│   ├── project/
│   │   ├── Dockerfile
│   │   └── entrypoint.sh
│   └── dev-stacks/
│       └── compose.yml          # PostgreSQL (+ MinIO for file-service)
├── resources/
│   ├── src/
│   │   ├── config/
│   │   │   └── settings.py      # Pydantic BaseSettings
│   │   ├── db/
│   │   │   ├── session.py       # async engine + get_session dependency
│   │   │   └── base.py          # SQLModel metadata + Base
│   │   ├── redis/
│   │   │   └── client.py        # shared Redis connection
│   │   ├── models/
│   │   │   └── <entity>.py      # SQLModel table models
│   │   ├── schemas/
│   │   │   └── <entity>.py      # Pydantic request/response schemas
│   │   ├── routers/
│   │   │   └── <entity>.py      # FastAPI APIRouter
│   │   ├── controllers/
│   │   │   └── <entity>.py      # Business logic (called by routers)
│   │   ├── middlewares/
│   │   │   ├── request_id.py    # X-Request-ID injection
│   │   │   ├── logging.py       # Structured request logging
│   │   │   └── service_auth.py  # X-Service-Token validation (inbound s2s)
│   │   ├── events/
│   │   │   ├── publisher.py     # EventPublisher helper
│   │   │   └── consumers/
│   │   │       └── <stream>.py  # One file per consumed stream
│   │   ├── libs/
│   │   │   ├── pagination.py    # PagedResponse + deps
│   │   │   ├── errors.py        # Standard error response + handlers
│   │   │   ├── s2s_client.py    # Outbound s2s HTTP client
│   │   │   └── health.py        # Liveness / readiness probes
│   │   └── tasks/
│   │       └── <task>.py        # Background tasks (APScheduler or Ofelia-driven)
│   ├── migrations/
│   │   ├── env.py
│   │   ├── script.py.mako
│   │   └── versions/
│   ├── tests/
│   │   ├── conftest.py
│   │   ├── unit/
│   │   └── integration/
│   └── server.py                # App factory + lifespan
├── .docker/
│   ├── project/
│   │   ├── Dockerfile
│   │   └── entrypoint.sh
│   └── compose/
│       ├── app.compose.yml      # FastAPI service (profiles: services)
│       ├── db.compose.yml       # PostgreSQL (profiles: services, ui, monitoring)
│       └── redis.compose.yml    # Redis (profiles: services, ui, monitoring)
├── compose.yml                  # Root: Docker Compose include directives
├── .env                         # Credentials — gitignored
├── devbox.json                  # Reproducible dev shell (Nix-backed)
├── .envrc                       # direnv — auto-activates devbox env
├── .python-version              # Python pin for uv: "3.13"
├── taskfile.yml                 # Root: includes sub-files + top-level aliases
├── taskfiles/
│   ├── dev.yml                  # Dev server
│   ├── test.yml                 # Pytest
│   ├── db.yml                   # Alembic migrations
│   ├── docker.yml               # Docker Compose
│   ├── deps.yml                 # uv dependency management
│   └── lint.yml                 # Ruff
├── pyproject.toml               # Project metadata + dependencies (uv)
├── uv.lock                      # Locked dependency graph (commit this)
└── README.md
```

---

## `server.py` — app factory and lifespan

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from src.config.settings import settings
from src.db.session import engine
from src.redis.client import get_redis
from src.middlewares.request_id import RequestIdMiddleware
from src.middlewares.logging import LoggingMiddleware
from src.libs.errors import register_exception_handlers
from src.libs.health import health_router
from src.events.consumers import start_all_consumers
from src.routers import router


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ── startup ──────────────────────────────────────────────
    redis = await get_redis()
    app.state.redis = redis
    await start_all_consumers(redis)   # bootstrap consumer groups + worker tasks
    yield
    # ── shutdown ─────────────────────────────────────────────
    await redis.aclose()


def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.SERVICE_NAME,
        version=settings.SERVICE_VERSION,
        docs_url="/docs" if settings.DEBUG else None,
    )

    # Middleware — applied bottom-up (last added = outermost)
    app.add_middleware(LoggingMiddleware)
    app.add_middleware(RequestIdMiddleware)

    register_exception_handlers(app)

    app.include_router(health_router)
    app.include_router(router, prefix="/api/v1")

    return app


app = create_app()
```

---

## Config — `src/config/settings.py`

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    # Identity
    SERVICE_NAME: str
    SERVICE_VERSION: str = "0.1.0"
    SERVICE_ID: str          # registered in auth-service service_clients
    SERVICE_SECRET: str      # bcrypt-hashed in auth-service DB

    # Database
    DATABASE_URL: str        # postgresql+asyncpg://...

    # Redis
    REDIS_URL: str           # redis://redis:6379/0

    # Auth
    AUTH_SERVICE_URL: str    # http://auth-service:8080
    RS256_PUBLIC_KEY: str    # PEM, validated locally — no auth-service call in hot path

    # Runtime
    APP_ENV: str = "production"   # "development" enables Uvicorn --reload
    WORKERS: int = 4              # Gunicorn worker count (prod only)
    DEBUG: bool = False
    LOG_LEVEL: str = "INFO"


settings = Settings()
```

---

## Database — `src/db/session.py`

```python
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from src.config.settings import settings

engine = create_async_engine(
    settings.DATABASE_URL,
    pool_size=10,
    max_overflow=20,
    echo=settings.DEBUG,
)

AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)


async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionLocal() as session:
        yield session
```

### Model base — `src/db/base.py`

```python
from sqlmodel import SQLModel

# Import all models here so Alembic autogenerate sees them
from src.models import *  # noqa: F401, F403

Base = SQLModel.metadata
```

### Model conventions

```python
# src/models/user.py
import uuid
from datetime import datetime
from sqlmodel import SQLModel, Field


class UserProfile(SQLModel, table=True):
    __tablename__ = "user_profiles"

    id:         uuid.UUID  = Field(default_factory=uuid.uuid4, primary_key=True)
    created_at: datetime   = Field(default_factory=datetime.utcnow, nullable=False)
    updated_at: datetime   = Field(default_factory=datetime.utcnow, nullable=False,
                                   sa_column_kwargs={"onupdate": datetime.utcnow})
```

---

## Redis — `src/redis/client.py`

```python
from redis.asyncio import Redis, from_url
from src.config.settings import settings

_client: Redis | None = None


async def get_redis() -> Redis:
    global _client
    if _client is None:
        _client = await from_url(settings.REDIS_URL, decode_responses=True)
    return _client
```

---

## Middleware stack

Order (outermost → innermost, i.e. bottom-up in `add_middleware` calls):

```
Request ──▶ RequestIdMiddleware ──▶ LoggingMiddleware ──▶ Route handler
```

### `RequestIdMiddleware`

```python
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request


class RequestIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        req_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
        request.state.request_id = req_id
        response = await call_next(request)
        response.headers["X-Request-ID"] = req_id
        return response
```

### `LoggingMiddleware`

```python
import time, structlog
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

logger = structlog.get_logger()


class LoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        start = time.perf_counter()
        response = await call_next(request)
        duration_ms = round((time.perf_counter() - start) * 1000, 2)
        logger.info(
            "http_request",
            method=request.method,
            path=request.url.path,
            status=response.status_code,
            duration_ms=duration_ms,
            request_id=getattr(request.state, "request_id", None),
        )
        return response
```

---

## Error handling — `src/libs/errors.py`

### Standard error envelope

All errors return the same JSON shape:

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "No user with id 'abc123'",
    "request_id": "f47ac10b-..."
  }
}
```

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel


class ErrorDetail(BaseModel):
    code: str
    message: str
    request_id: str | None = None


class ErrorResponse(BaseModel):
    error: ErrorDetail


class AppError(Exception):
    def __init__(self, code: str, message: str, status_code: int = 400):
        self.code = code
        self.message = message
        self.status_code = status_code


def register_exception_handlers(app: FastAPI) -> None:
    @app.exception_handler(AppError)
    async def app_error_handler(request: Request, exc: AppError):
        return JSONResponse(
            status_code=exc.status_code,
            content={"error": {
                "code": exc.code,
                "message": exc.message,
                "request_id": getattr(request.state, "request_id", None),
            }},
        )

    @app.exception_handler(Exception)
    async def unhandled_error_handler(request: Request, exc: Exception):
        return JSONResponse(
            status_code=500,
            content={"error": {
                "code": "INTERNAL_ERROR",
                "message": "An unexpected error occurred.",
                "request_id": getattr(request.state, "request_id", None),
            }},
        )
```

---

## Pagination — `src/libs/pagination.py`

```python
from typing import Generic, TypeVar
from pydantic import BaseModel
from fastapi import Query

T = TypeVar("T")


class PagedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    page_size: int
    pages: int


def pagination_params(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
) -> tuple[int, int]:
    return page, page_size


def paginate(items: list[T], total: int, page: int, page_size: int) -> PagedResponse[T]:
    return PagedResponse(
        items=items,
        total=total,
        page=page,
        page_size=page_size,
        pages=-(-total // page_size),  # ceil division
    )
```

---

## Health checks — `src/libs/health.py`

Two probes — standard Kubernetes/Docker convention:

| Endpoint            | Purpose               | Fails when                 |
| ------------------- | --------------------- | -------------------------- |
| `GET /health/live`  | Is the process alive? | Never (process is running) |
| `GET /health/ready` | Can it serve traffic? | DB or Redis unreachable    |

```python
from fastapi import APIRouter
from fastapi.responses import JSONResponse
from sqlalchemy import text
from src.db.session import AsyncSessionLocal
from src.redis.client import get_redis

health_router = APIRouter(tags=["health"])


@health_router.get("/health/live")
async def liveness():
    return {"status": "ok"}


@health_router.get("/health/ready")
async def readiness():
    checks: dict[str, str] = {}

    # PostgreSQL
    try:
        async with AsyncSessionLocal() as s:
            await s.execute(text("SELECT 1"))
        checks["db"] = "ok"
    except Exception:
        checks["db"] = "error"

    # Redis
    try:
        redis = await get_redis()
        await redis.ping()
        checks["redis"] = "ok"
    except Exception:
        checks["redis"] = "error"

    healthy = all(v == "ok" for v in checks.values())
    return JSONResponse(
        status_code=200 if healthy else 503,
        content={"status": "ready" if healthy else "degraded", "checks": checks},
    )
```

---

## Event publishing — `src/events/publisher.py`

```python
from datetime import datetime, timezone
from redis.asyncio import Redis
from src.config.settings import settings


class EventPublisher:
    def __init__(self, redis: Redis):
        self._redis = redis

    async def publish(
        self,
        stream: str,
        payload: dict[str, str],
        *,
        maxlen: int = 50_000,
        issuer_service: str | None = None,
        issued_to_service: str | None = None,
    ) -> None:
        envelope: dict[str, str] = {
            "source_service": settings.SERVICE_ID,
            "timestamp":      datetime.now(timezone.utc).isoformat(),
            **payload,
        }
        if issuer_service:
            envelope["issuer_service"] = issuer_service
        if issued_to_service:
            envelope["issued_to_service"] = issued_to_service

        await self._redis.xadd(stream, envelope, maxlen=maxlen, approximate=True)
```

Usage in a controller:

```python
await publisher.publish(
    "user.profile.updated",
    {"user_id": str(user.id), "fields_changed": "display_name,avatar"},
)
```

---

## Event consuming — `src/events/consumers/<stream>.py`

```python
# src/events/consumers/auth_user_registered.py
import asyncio, socket
from redis.asyncio import Redis
from redis.exceptions import ResponseError
import structlog

STREAM   = "auth.user.registered"
GROUP    = settings.SERVICE_ID           # e.g. "user-service"
WORKER   = f"{settings.SERVICE_ID}-{socket.gethostname()}"
BLOCK_MS = 5_000
RECLAIM_AFTER_MS = 60_000

logger = structlog.get_logger()


async def bootstrap(redis: Redis) -> None:
    try:
        await redis.xgroup_create(STREAM, GROUP, id="0", mkstream=True)
    except ResponseError:
        pass  # already exists


async def handle(msg_id: str, data: dict[str, str]) -> None:
    # ── business logic ───────────────────────────────────────
    user_id = data["user_id"]
    logger.info("user_registered_event", user_id=user_id)
    # ...


async def run(redis: Redis) -> None:
    await bootstrap(redis)
    while True:
        # Reclaim idle messages from crashed peers
        claimed, *_ = await redis.xautoclaim(
            STREAM, GROUP, WORKER, min_idle_time=RECLAIM_AFTER_MS, count=10
        )
        for msg_id, data in claimed:
            try:
                await handle(msg_id, data)
                await redis.xack(STREAM, GROUP, msg_id)
            except Exception:
                logger.exception("reclaim_handle_failed", msg_id=msg_id)

        # Read new messages
        results = await redis.xreadgroup(
            GROUP, WORKER, {STREAM: ">"}, count=10, block=BLOCK_MS
        )
        for _, messages in (results or []):
            for msg_id, data in messages:
                try:
                    await handle(msg_id, data)
                    await redis.xack(STREAM, GROUP, msg_id)
                except Exception:
                    logger.exception("handle_failed", msg_id=msg_id)
                    # Leave in PEL for reclaim on next cycle
```

### `src/events/consumers/__init__.py`

```python
import asyncio
from redis.asyncio import Redis
from src.events.consumers import auth_user_registered  # import all consumers

CONSUMERS = [auth_user_registered]  # add each consumer module here


async def start_all_consumers(redis: Redis) -> None:
    for consumer in CONSUMERS:
        asyncio.create_task(consumer.run(redis))
```

---

## Service-to-service client — `src/libs/s2s_client.py`

Thin wrapper around `httpx.AsyncClient` that automatically attaches a valid
service JWT. See [service-auth.md](./service-auth.md) for the full token
lifecycle.

```python
import httpx
from src.config.settings import settings
# ServiceTokenCache is documented in service-auth.md
from src.libs.service_token_cache import ServiceTokenCache


class S2SClient:
    def __init__(self, base_url: str, token_cache: ServiceTokenCache):
        self._base = base_url
        self._cache = token_cache

    async def get(self, path: str, **kwargs) -> httpx.Response:
        return await self._request("GET", path, **kwargs)

    async def post(self, path: str, **kwargs) -> httpx.Response:
        return await self._request("POST", path, **kwargs)

    async def _request(self, method: str, path: str, **kwargs) -> httpx.Response:
        token = await self._cache.get()
        headers = kwargs.pop("headers", {})
        headers["X-Service-Token"] = token
        async with httpx.AsyncClient(base_url=self._base) as client:
            return await client.request(method, path, headers=headers, **kwargs)
```

---

## Observability

### Structured logging — `structlog`

Configure once at startup in `server.py`:

```python
import logging, structlog

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
    wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
    logger_factory=structlog.PrintLoggerFactory(),
)
```

Every log line is newline-delimited JSON — consumed by Promtail → Loki.

### Prometheus metrics

Mount the `/metrics` endpoint via `prometheus-fastapi-instrumentator`:

```python
from prometheus_fastapi_instrumentator import Instrumentator

Instrumentator().instrument(app).expose(app, endpoint="/metrics")
```

Scraped by Prometheus; visualised in Grafana. Add custom counters per service:

```python
from prometheus_client import Counter

TOKEN_FAILURES = Counter(
    "service_token_validation_failures_total",
    "Number of rejected inbound service JWTs",
    ["reason"],
)
# Usage:
TOKEN_FAILURES.labels(reason="expired").inc()
```

---

## Testing

### Stack

| Layer           | Tool                                           |
| --------------- | ---------------------------------------------- |
| Test runner     | `pytest` + `pytest-asyncio`                    |
| Async fixtures  | `anyio`                                        |
| DB isolation    | `testcontainers-postgres` (real PG, not mocks) |
| Redis isolation | `testcontainers-redis`                         |
| HTTP mocking    | `respx` (for s2s calls)                        |
| Coverage        | `pytest-cov`                                   |

### `tests/conftest.py`

```python
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from src.server import create_app
from src.db.base import Base


@pytest_asyncio.fixture(scope="session")
async def pg():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg.get_connection_url().replace("psycopg2", "asyncpg")


@pytest_asyncio.fixture(scope="session")
async def redis_url():
    with RedisContainer("redis:7-alpine") as r:
        yield f"redis://{r.get_container_host_ip()}:{r.get_exposed_port(6379)}/0"


@pytest_asyncio.fixture(scope="function")
async def db_session(pg):
    engine = create_async_engine(pg)
    async with engine.begin() as conn:
        await conn.run_sync(Base.create_all)
    session_factory = async_sessionmaker(engine, expire_on_commit=False)
    async with session_factory() as session:
        yield session
    async with engine.begin() as conn:
        await conn.run_sync(Base.drop_all)


@pytest_asyncio.fixture(scope="function")
async def client(db_session, redis_url, monkeypatch):
    monkeypatch.setenv("DATABASE_URL", str(db_session.bind.url))
    monkeypatch.setenv("REDIS_URL", redis_url)
    app = create_app()
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
```

### Test conventions

```python
# tests/integration/test_profile.py
async def test_get_profile_returns_404_for_unknown_user(client):
    resp = await client.get("/api/v1/users/00000000-0000-0000-0000-000000000000")
    assert resp.status_code == 404
    assert resp.json()["error"]["code"] == "USER_NOT_FOUND"
```

---

## Dockerfile & `entrypoint.sh`

### Dockerfile — multi-stage (with uv)

```dockerfile
# ---- build stage -------------------------------------------------------
# uv image with Python baked in — tag is the version pin
FROM ghcr.io/astral-sh/uv:python3.13-slim AS builder

ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PROJECT_ENVIRONMENT=/app/.venv

WORKDIR /app

# Install dependencies first (cached layer — only busts on pyproject.toml / uv.lock change)
COPY pyproject.toml uv.lock .python-version ./
RUN uv sync --frozen --no-dev --no-install-project

# Copy source and finalize install
COPY resources/ resources/
RUN uv sync --frozen --no-dev

# ---- runtime stage -----------------------------------------------------
FROM python:3.13-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/app/.venv/bin:$PATH"

WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY --from=builder /app/resources /app/resources
COPY .docker/project/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 8080
ENTRYPOINT ["/entrypoint.sh"]
```

### `.docker/project/entrypoint.sh`

The entrypoint always runs migrations first, then picks the server based on
`APP_ENV`.

- **development** → Uvicorn with `--reload` (live reload, single worker)
- **production** (default) → Gunicorn with `UvicornWorker` (multi-worker, no reload)

```sh
#!/bin/sh
set -e

# Run pending migrations before starting the server
alembic -c resources/migrations/alembic.ini upgrade head

if [ "${APP_ENV:-production}" = "development" ]; then
    exec uvicorn resources.server:app \
        --reload \
        --host 0.0.0.0 \
        --port 8080
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

> **Worker count** — rule of thumb: `WORKERS = 2 × num_vCPU + 1`.  
> Override via the `WORKERS` env var in your deployment config.  
> The `UvicornWorker` keeps full async support (event loop per worker).

### `.python-version`

Read by `uv` automatically — pins the interpreter for local dev and inside Docker.

```
3.13
```

### Required `pyproject.toml`

```toml
[project]
name = "<service-name>"
version = "0.1.0"
description = "<one-line description>"
requires-python = ">=3.13"
dependencies = [
    "fastapi>=0.115",
    "uvicorn[standard]>=0.30",
    "gunicorn>=22.0",
    "sqlmodel>=0.0.21",
    "asyncpg>=0.29",
    "alembic>=1.13",
    "redis>=5.0",
    "pyjwt[crypto]>=2.8",
    "pydantic-settings>=2.0",
    "structlog>=24.0",
    "prometheus-fastapi-instrumentator>=7.0",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-cov>=5.0",
    "httpx>=0.27",
    "testcontainers[postgres,redis]>=4.0",
    "ruff>=0.4",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["resources/tests"]

[tool.ruff.lint]
select = ["E", "F", "I", "UP"]

[tool.ruff.format]
quote-style = "double"
```

> `[dependency-groups]` is the PEP 735 standard used by uv for dev/optional groups.
> Run `uv sync` (with no flags) to install everything including dev deps.
> Run `uv sync --no-dev` inside Docker to install prod deps only.

---

---

## Compose files — dev stack

Compose is split into three fragments under `.docker/compose/`, each with
**profiles** mirroring the [dev-toolkit](https://github.com/cyboooooorg/dev-toolkit) pattern:

| Profile | What it adds |
|---|---|
| `services` | Core service — always on |
| `ui` | Admin UI (pgAdmin / RedisInsight) — opt-in |
| `monitoring` | Metrics exporter — opt-in |

The root `compose.yml` uses Docker Compose `include:` to pull all three fragments.
All credentials come from `.env` — never hardcoded.

### `compose.yml` — root

```yaml
# Root compose — includes all service fragments.
# Run with: docker compose up -d
# Opt-in profiles: --profile ui   --profile monitoring

include:
  - path: .docker/compose/app.compose.yml
  - path: .docker/compose/db.compose.yml
  - path: .docker/compose/redis.compose.yml
```

### `.docker/compose/app.compose.yml`

```yaml
# FastAPI application service
# Credentials: env_file .env — never hardcoded

services:
  <service-name>:
    build:
      context: .
      dockerfile: .docker/project/Dockerfile
    profiles:
      - services
    container_name: ${COMPOSE_PROJECT_NAME}-app
    env_file: .env
    ports:
      - "127.0.0.1:${SERVICE_PORT}:8080"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app_net
      - evoframe-internal
    labels:
      ofelia.enabled: "true"    # see plan/infra/ofelia.md
    restart: unless-stopped

networks:
  app_net:
    name: ${COMPOSE_PROJECT_NAME}_app
  evoframe-internal:
    name: evoframe-internal
    external: true
```

### `.docker/compose/db.compose.yml`

```yaml
# PostgreSQL — one isolated DB per service
# Credentials: env_file .env — never hardcoded

services:
  db:
    image: postgres:16-alpine
    profiles:
      - services
    container_name: ${COMPOSE_PROJECT_NAME}-db
    env_file: .env
    environment:
      POSTGRES_USER:     ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB:       ${DB_NAME}
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "${DB_USER}", "-d", "${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - db_net
      - evoframe-internal
    restart: unless-stopped

  pgadmin:
    image: dpage/pgadmin4:latest
    profiles:
      - ui
    container_name: ${COMPOSE_PROJECT_NAME}-pgadmin
    ports:
      - "127.0.0.1:${PGADMIN_PORT}:80"
    environment:
      PGADMIN_DEFAULT_EMAIL:    ${PGADMIN_EMAIL}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_PASSWORD}
    # No volume — ephemeral by design
    networks:
      - db_net
    restart: unless-stopped

  db_exporter:
    image: prometheuscommunity/postgres-exporter:latest
    profiles:
      - monitoring
    container_name: ${COMPOSE_PROJECT_NAME}-db-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}?sslmode=disable"
    # No volume — ephemeral by design
    networks:
      - db_net
      - evoframe-internal
    restart: unless-stopped

volumes:
  db_data:

networks:
  db_net:
    name: ${COMPOSE_PROJECT_NAME}_db
  evoframe-internal:
    name: evoframe-internal
    external: true
```

### `.docker/compose/redis.compose.yml`

```yaml
# Redis — shared event bus + cache (per-service dev instance)
# Credentials: env_file .env — never hardcoded

services:
  redis:
    image: redis:7-alpine
    profiles:
      - services
    container_name: ${COMPOSE_PROJECT_NAME}-redis
    env_file: .env
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - redis_net
      - evoframe-internal
    restart: unless-stopped

  redisinsight:
    image: redis/redisinsight:latest
    profiles:
      - ui
    container_name: ${COMPOSE_PROJECT_NAME}-redisinsight
    ports:
      - "127.0.0.1:${REDIS_UI_PORT}:5540"
    environment:
      RI_REDIS_HOST:     redis
      RI_REDIS_PORT:     6379
      RI_REDIS_PASSWORD: ${REDIS_PASSWORD}
    # No volume — ephemeral by design
    networks:
      - redis_net
    restart: unless-stopped

  redis_exporter:
    image: oliver006/redis_exporter:latest
    profiles:
      - monitoring
    container_name: ${COMPOSE_PROJECT_NAME}-redis-exporter
    environment:
      REDIS_ADDR:     "redis:6379"
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    # No volume — ephemeral by design
    networks:
      - redis_net
      - evoframe-internal
    restart: unless-stopped

volumes:
  redis_data:

networks:
  redis_net:
    name: ${COMPOSE_PROJECT_NAME}_redis
  evoframe-internal:
    name: evoframe-internal
    external: true
```

---

## `devbox.json` — reproducible dev shell

Devbox uses Nix under the hood to provide a pinned, isolated shell with all
tools pre-installed. No system-wide installs required.

```json
{
  "$schema": "https://raw.githubusercontent.com/jetify-com/devbox/0.13.7/.schema/devbox.schema.json",
  "packages": ["python@3.13", "uv@latest", "go-task@latest", "docker@latest"],
  "shell": {
    "init_hook": ["uv sync"]
  },
  "env": {
    "APP_ENV": "development"
  }
}
```

Generate a `.envrc` for direnv auto-activation:

```sh
devbox generate direnv
direnv allow
```

After that, entering the project directory auto-activates the devbox shell and
runs `uv sync`.

---

## `taskfile.yml` — root (includes + aliases)

The root file includes the six sub-taskfiles and exposes short top-level aliases.
The guard lives in each sub-file — every task is protected regardless of how it's called.

```yaml
version: "3"

includes:
  dev:    ./taskfiles/dev.yml
  test:   ./taskfiles/test.yml
  db:     ./taskfiles/db.yml
  docker: ./taskfiles/docker.yml
  deps:   ./taskfiles/deps.yml
  lint:   ./taskfiles/lint.yml

tasks:
  dev:
    desc: "▶  Start dev server (watch mode)"
    cmd: task dev:start

  test:
    desc: "🧪  Run full test suite"
    cmd: task test:run

  migrate:
    desc: "🗄  Apply pending DB migrations"
    cmd: task db:migrate

  up:
    desc: "🐳  Start full dev stack (app + db + redis)"
    cmd: task docker:up

  up-ui:
    desc: "🐳  Start full stack + admin UIs"
    cmd: task docker:up-ui

  up-monitoring:
    desc: "🐳  Start full stack + monitoring exporters"
    cmd: task docker:up-monitoring

  down:
    desc: "🐳  Stop dev stack"
    cmd: task docker:down

  logs:
    desc: "📋  Follow container logs"
    cmd: task docker:logs

  lint:
    desc: "🔍  Run linter"
    cmd: task lint:check

  format:
    desc: "✨  Auto-format code"
    cmd: task lint:format

  sync:
    desc: "📦  Sync deps from uv.lock"
    cmd: task deps:sync
```

---

## `taskfiles/dev.yml`

```yaml
version: "3"

tasks:
  _guard:
    internal: true
    preconditions:
      - sh: test -n "$DEVBOX_SHELL_ENABLED"
        msg: "Not in devbox shell. Run: devbox shell  or  direnv allow"

  start:
    desc: Start Uvicorn with --reload
    deps: [_guard]
    cmd: uv run uvicorn resources.server:app --reload --host 0.0.0.0 --port 8080
```

---

## `taskfiles/test.yml`

```yaml
version: "3"

tasks:
  _guard:
    internal: true
    preconditions:
      - sh: test -n "$DEVBOX_SHELL_ENABLED"
        msg: "Not in devbox shell. Run: devbox shell  or  direnv allow"

  run:
    desc: Run full test suite with coverage
    deps: [_guard]
    cmd: uv run pytest resources/tests --cov=resources/src --cov-report=term-missing

  unit:
    desc: Run unit tests only
    deps: [_guard]
    cmd: uv run pytest resources/tests/unit -v

  integration:
    desc: Run integration tests only
    deps: [_guard]
    cmd: uv run pytest resources/tests/integration -v
```

---

## `taskfiles/db.yml`

```yaml
version: "3"

tasks:
  _guard:
    internal: true
    preconditions:
      - sh: test -n "$DEVBOX_SHELL_ENABLED"
        msg: "Not in devbox shell. Run: devbox shell  or  direnv allow"

  migrate:
    desc: Apply pending Alembic migrations
    deps: [_guard]
    cmd: uv run alembic -c resources/migrations/alembic.ini upgrade head

  new:
    desc: Generate a new migration (name=<name>)
    deps: [_guard]
    cmd: uv run alembic -c resources/migrations/alembic.ini revision --autogenerate -m "{{.name}}"

  rollback:
    desc: Rollback one migration step
    deps: [_guard]
    cmd: uv run alembic -c resources/migrations/alembic.ini downgrade -1

  history:
    desc: Show full migration history
    deps: [_guard]
    cmd: uv run alembic -c resources/migrations/alembic.ini history --verbose
```

---

## `taskfiles/docker.yml`

```yaml
version: "3"

vars:
  COMPOSE_APP: .docker/compose/app.compose.yml
  COMPOSE_DB:  .docker/compose/db.compose.yml
  COMPOSE_RED: .docker/compose/redis.compose.yml

tasks:
  _guard:
    internal: true
    preconditions:
      - sh: test -n "$DEVBOX_SHELL_ENABLED"
        msg: "Not in devbox shell. Run: devbox shell  or  direnv allow"

  # ---- full stack ------------------------------------------------------
  up:
    desc: Start full dev stack (app + db + redis)
    deps: [_guard]
    cmd: docker compose up -d --remove-orphans

  up-ui:
    desc: Start full stack + all UI companions (pgAdmin, RedisInsight)
    deps: [_guard]
    cmd: docker compose --profile ui up -d --remove-orphans

  up-monitoring:
    desc: Start full stack + all monitoring exporters
    deps: [_guard]
    cmd: docker compose --profile monitoring up -d --remove-orphans

  down:
    desc: Stop and remove all containers
    deps: [_guard]
    cmd: docker compose down

  restart:
    desc: Restart all services
    deps: [_guard]
    cmd: docker compose restart

  build:
    desc: Rebuild the app image
    deps: [_guard]
    cmd: docker compose build

  ps:
    desc: List running containers and their status
    deps: [_guard]
    cmd: docker compose ps

  logs:
    desc: Follow logs (SERVICE=<name> for one container, default all)
    deps: [_guard]
    cmd: docker compose logs -f {{.SERVICE}}

  clean:
    desc: "⚠  Stop containers and delete all volumes (destructive)"
    deps: [_guard]
    cmd: docker compose down -v --remove-orphans

  # ---- app only --------------------------------------------------------
  app-up:
    desc: Start app service only
    deps: [_guard]
    cmd: docker compose -f {{.COMPOSE_APP}} --profile services up -d --remove-orphans

  app-logs:
    desc: Follow app container logs
    deps: [_guard]
    cmd: docker compose -f {{.COMPOSE_APP}} logs -f

  app-restart:
    desc: Restart app container
    deps: [_guard]
    cmd: docker compose -f {{.COMPOSE_APP}} restart

  # ---- db only ---------------------------------------------------------
  db-up:
    desc: Start PostgreSQL only
    deps: [_guard]
    cmd: docker compose -f {{.COMPOSE_DB}} --profile services up -d --remove-orphans

  db-ui:
    desc: Start PostgreSQL + pgAdmin UI
    deps: [_guard]
    cmd: docker compose -f {{.COMPOSE_DB}} --profile services --profile ui up -d --remove-orphans

  db-logs:
    desc: Follow PostgreSQL container logs
    deps: [_guard]
    cmd: docker compose -f {{.COMPOSE_DB}} logs -f db

  db-restart:
    desc: Restart PostgreSQL container
    deps: [_guard]
    cmd: docker compose -f {{.COMPOSE_DB}} restart db

  # ---- redis only ------------------------------------------------------
  redis-up:
    desc: Start Redis only
    deps: [_guard]
    cmd: docker compose -f {{.COMPOSE_RED}} --profile services up -d --remove-orphans

  redis-ui:
    desc: Start Redis + RedisInsight UI
    deps: [_guard]
    cmd: docker compose -f {{.COMPOSE_RED}} --profile services --profile ui up -d --remove-orphans

  redis-logs:
    desc: Follow Redis container logs
    deps: [_guard]
    cmd: docker compose -f {{.COMPOSE_RED}} logs -f redis

  redis-restart:
    desc: Restart Redis container
    deps: [_guard]
    cmd: docker compose -f {{.COMPOSE_RED}} restart redis
```

---

## `taskfiles/deps.yml`

```yaml
version: "3"

tasks:
  _guard:
    internal: true
    preconditions:
      - sh: test -n "$DEVBOX_SHELL_ENABLED"
        msg: "Not in devbox shell. Run: devbox shell  or  direnv allow"

  sync:
    desc: Sync deps from uv.lock (installs missing, removes extra)
    deps: [_guard]
    cmd: uv sync

  add:
    desc: Add a runtime dependency (name=<package>)
    deps: [_guard]
    cmd: uv add {{.name}}

  add-dev:
    desc: Add a dev dependency (name=<package>)
    deps: [_guard]
    cmd: uv add --group dev {{.name}}

  update:
    desc: Upgrade all deps and regenerate uv.lock
    deps: [_guard]
    cmd: uv lock --upgrade
```

---

## `taskfiles/lint.yml`

```yaml
version: "3"

tasks:
  _guard:
    internal: true
    preconditions:
      - sh: test -n "$DEVBOX_SHELL_ENABLED"
        msg: "Not in devbox shell. Run: devbox shell  or  direnv allow"

  check:
    desc: Run ruff linter (report only)
    deps: [_guard]
    cmd: uv run ruff check resources/src resources/tests

  format:
    desc: Run ruff formatter
    deps: [_guard]
    cmd: uv run ruff format resources/src resources/tests

  fix:
    desc: Auto-fix lint issues in place
    deps: [_guard]
    cmd: uv run ruff check --fix resources/src resources/tests
```

---

## Checklist — new service scaffold

- [ ] Copy folder structure from blueprint
- [ ] Run `devbox install` then `devbox generate direnv && direnv allow`
- [ ] Copy `pyproject.toml`, update `name` and `description`, run `uv sync`
- [ ] Fill in `src/config/settings.py` with service-specific env vars
- [ ] Register service identity in `auth-service` `service_clients` table
- [ ] Create initial Alembic migration (`task migrate-new name=init`)
- [ ] Implement `/health/live` and `/health/ready`
- [ ] Add `X-Service-Token` validation middleware if service accepts s2s calls
- [ ] Register consumer groups for all subscribed streams in `start_all_consumers`
- [ ] Add `ofelia.job-run.*` labels for any scheduled tasks
- [ ] Add service to global `compose.yml`
- [ ] Add service to `plan/README.md` service index
