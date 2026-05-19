# EvoFrame — Architecture Diagrams

## 1. Service Topology

Full platform view showing all services, infrastructure, and external dependencies.

```mermaid
graph TB
    classDef external  fill:#dbeafe,stroke:#3b82f6,color:#1e3a8a
    classDef edge      fill:#fef3c7,stroke:#f59e0b,color:#78350f
    classDef core      fill:#dcfce7,stroke:#22c55e,color:#14532d
    classDef support   fill:#f3e8ff,stroke:#a855f7,color:#581c87
    classDef infra     fill:#f1f5f9,stroke:#94a3b8,color:#1e293b

    Client([" Client<br/>(Browser / Mobile / API) "]):::external
    Stripe([" Stripe<br/>(Payments) "]):::external
    SMTP([" SMTP Provider<br/>(Email) "]):::external

    subgraph Edge["Edge Layer"]
        GW["🔀 api-gateway<br/>Routing · JWT validation<br/>Rate limiting · CORS"]:::edge
    end

    subgraph Core["Core Services"]
        direction TB
        AS["🔐 auth-service<br/>Auth · Authz · JWT (RS256)<br/>RBAC · MFA · OAuth2"]:::core
        US["👤 user-service<br/>Profiles · Preferences<br/>Avatar"]:::core
        TS["🏢 team-service<br/>Teams · Members<br/>Invitations"]:::core
        BS["💳 billing-service<br/>Subscriptions · Invoices<br/>Stripe webhooks"]:::core
        NS["🔔 notification-service<br/>Email · In-app inbox<br/>Event consumer"]:::core
        FS["📁 file-service<br/>Upload · Presigned URLs<br/>S3 · MinIO"]:::core
        AUS["📋 audit-service<br/>Immutable event log<br/>Compliance (read-only API)"]:::core
    end

    subgraph Infra["Infrastructure"]
        direction LR
        Redis[("Redis<br/>Streams + Cache<br/>Sessions")]:::infra
        PG[("PostgreSQL<br/>× 7 isolated instances<br/>one per service")]:::infra
        MinIO[("MinIO<br/>Object Storage<br/>(dev)")]:::infra
        Prom["Prometheus<br/>+ Grafana"]:::infra
    end

    Client -->|"HTTPS"| GW
    GW -->|"introspect (every request)"| AS
    GW --> US
    GW --> TS
    GW --> BS
    GW --> NS
    GW --> FS
    GW --> AUS

    BS -->|"Stripe API"| Stripe
    NS -->|"SMTP"| SMTP
    FS -->|"S3 API"| MinIO

    AS --->|"publish events"| Redis
    US --->|"publish events"| Redis
    TS --->|"publish events"| Redis
    BS --->|"publish events"| Redis
    FS --->|"publish events"| Redis
    NS -.->|"consume events"| Redis
    AUS -.->|"consume ALL events"| Redis

    Core ---|"each owns its own DB"| PG
    Core -.-|"metrics"| Prom
```

---

## 2. Request / Authentication Flow

How a typical authenticated API request flows through the platform.

```mermaid
sequenceDiagram
    actor Client
    participant GW as api-gateway
    participant AS as auth-service
    participant SVC as target service<br/>(e.g. user-service)
    participant Redis

    Note over Client,Redis: Login (one-time)
    Client->>GW: POST /auth/login {email, password}
    GW->>AS: proxy request
    AS->>AS: verify credentials (Argon2)
    AS->>Redis: store refresh_token (hashed, TTL 30d)
    AS-->>GW: { access_token (RS256 JWT 15min), refresh_token }
    GW-->>Client: tokens

    Note over Client,Redis: Subsequent authenticated request
    Client->>GW: GET /users/me (Authorization: Bearer token)
    GW->>AS: GET /auth/introspect (Authorization: Bearer token)
    AS->>AS: verify RS256 signature + expiry
    AS-->>GW: { valid: true, user_id, roles, team_id }
    GW->>SVC: GET /users/me (X-User-Id, X-User-Roles, X-Internal-Token)
    SVC->>SVC: business logic
    SVC-->>GW: 200 { ...profile }
    GW-->>Client: 200 { ...profile }

    Note over Client,Redis: Token refresh
    Client->>GW: POST /auth/refresh { refresh_token }
    GW->>AS: proxy
    AS->>Redis: validate + rotate refresh_token
    AS-->>GW: { new access_token, new refresh_token }
    GW-->>Client: new tokens
```

---

## 3. Redis Streams — Event Bus

Which services publish and consume which event streams.

```mermaid
graph LR
    classDef pub  fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef sub  fill:#dbeafe,stroke:#3b82f6,color:#1e3a8a
    classDef bus  fill:#fef9c3,stroke:#ca8a04,color:#713f12

    subgraph Publishers
        GW_P["api-gateway"]:::pub
        AS_P["auth-service"]:::pub
        US_P["user-service"]:::pub
        TS_P["team-service"]:::pub
        BS_P["billing-service"]:::pub
        FS_P["file-service"]:::pub
    end

    subgraph Streams["Redis Streams (event bus)"]
        E0(["auth.access.denied"]):::bus
        E1(["auth.user.registered"]):::bus
        E2(["auth.user.password_reset"]):::bus
        E3(["auth.user.mfa_changed"]):::bus
        E4(["user.profile.updated"]):::bus
        E5(["user.account.deleted"]):::bus
        E6(["team.member.invited"]):::bus
        E7(["team.member.joined"]):::bus
        E8(["team.member.removed"]):::bus
        E9(["team.deleted"]):::bus
        E10(["billing.subscription.activated"]):::bus
        E11(["billing.subscription.cancelled"]):::bus
        E12(["billing.payment.failed"]):::bus
        E13(["file.uploaded"]):::bus
        E14(["file.deleted"]):::bus
    end

    subgraph Subscribers
        NS_S["notification-service"]:::sub
        AUS_S["audit-service"]:::sub
        US_S["user-service"]:::sub
        TS_S["team-service"]:::sub
        BS_S["billing-service"]:::sub
        FS_S["file-service"]:::sub
    end

    GW_P --> E0
    AS_P --> E1 & E2 & E3
    US_P --> E4 & E5
    TS_P --> E6 & E7 & E8 & E9
    BS_P --> E10 & E11 & E12
    FS_P --> E13 & E14

    E0 --> AUS_S
    E1 --> NS_S & AUS_S & US_S
    E2 --> NS_S & AUS_S
    E3 --> NS_S & AUS_S
    E4 --> AUS_S
    E5 --> TS_S & FS_S & AUS_S
    E6 --> NS_S & AUS_S
    E7 --> NS_S & AUS_S
    E8 --> NS_S & AUS_S
    E9 --> BS_S & AUS_S
    E10 --> NS_S & AUS_S
    E11 --> NS_S & AUS_S
    E12 --> NS_S & AUS_S
    E13 --> AUS_S
    E14 --> AUS_S
```

---

## 4. Security Layers

Defense-in-depth from the client to the database.

```mermaid
graph TB
    classDef layer fill:#f8fafc,stroke:#64748b,color:#1e293b
    classDef threat fill:#fee2e2,stroke:#ef4444,color:#7f1d1d

    L1["① TLS termination<br/>(nginx / Caddy in prod)"]:::layer
    L2["② CORS policy<br/>(allowed origins whitelist)"]:::layer
    L3["③ Rate limiting<br/>(per-IP + per-user, Redis counters)"]:::layer
    L4["④ JWT validation (RS256)<br/>(every request, api-gateway → auth-service)"]:::layer
    L5["⑤ X-Internal-Token<br/>(blocks direct service access from outside)"]:::layer
    L6["⑥ RBAC enforcement<br/>(per service, roles from X-User-Roles header)"]:::layer
    L7["⑦ Pydantic input validation<br/>(all request bodies, all services)"]:::layer
    L8["⑧ Minimum-privilege DB user<br/>(audit-service: INSERT-only)"]:::layer
    L9["⑨ Stripe HMAC verification<br/>(webhook signatures)"]:::layer
    L10["⑩ Presigned file URLs<br/>(short TTL, signed per user)"]:::layer

    L1 --> L2 --> L3 --> L4 --> L5 --> L6 --> L7 --> L8
    L4 -.-> L9
    L6 -.-> L10
```
