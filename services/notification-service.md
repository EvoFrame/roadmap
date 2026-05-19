# notification-service

> **Status:** 📋 Planned (Phase 6)  
> **Role:** Email and in-app notifications

## Description

Pure consumer service — it holds an in-app notification inbox per user and dispatches emails in response to platform events. Users control per-channel preferences. The service never originates events; it is a leaf in the event graph.

## Tech Stack

| Concern | Technology |
|---|---|
| Framework | FastAPI |
| Database | PostgreSQL (own instance) |
| Event consumption | Redis Streams (consumer group `notification-service`) |
| Email dispatch | `aiosmtplib` over SMTP (Resend / SendGrid SMTP relay) |
| Templating | Jinja2 (server-rendered, no user HTML) |

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/notifications` | Paginated in-app inbox |
| `PATCH` | `/notifications/{id}/read` | Mark notification as read |
| `PATCH` | `/notifications/read-all` | Mark all as read |
| `DELETE` | `/notifications/{id}` | Dismiss notification |
| `GET` | `/notifications/preferences` | Per-channel preferences |
| `PATCH` | `/notifications/preferences` | Update preferences |
| `GET` | `/notifications/unread-count` | Badge count |

## Events Consumed → Actions

| Stream | Email | In-app |
|---|---|---|
| `auth.user.registered` | Welcome email | — |
| `auth.user.password_reset` | Password reset link | — |
| `auth.user.mfa_changed` | MFA status changed | ✅ |
| `team.member.invited` | Invite email (carries token) | — |
| `team.member.joined` | Notify team admin | ✅ |
| `team.member.removed` | Notify removed user | ✅ |
| `billing.subscription.activated` | Subscription confirmed | ✅ |
| `billing.subscription.cancelled` | Cancellation confirmed | ✅ |
| `billing.payment.failed` | Payment failure alert | ✅ |

## Events Published

None — this is a leaf service.

## Data Model (key fields)

```
notifications
├── id          UUID PK
├── user_id     UUID
├── type        TEXT (e.g. 'team.member.joined')
├── title       TEXT
├── body        TEXT
├── is_read     BOOL default false
├── created_at  TIMESTAMPTZ
└── read_at     TIMESTAMPTZ nullable

notification_preferences
├── user_id         UUID PK
├── email_enabled   BOOL default true
├── inapp_enabled   BOOL default true
└── updated_at      TIMESTAMPTZ
```

## Security Notes

- Email templates are server-rendered Jinja2 — user-controlled content is escaped, never raw HTML
- Email rate limiting per user (max 10 transactional emails/hour) to prevent abuse
- Users only access their own notifications (enforced via `X-User-Id`)
- Consumer group ensures each event is processed exactly once (Redis Streams ACK)
