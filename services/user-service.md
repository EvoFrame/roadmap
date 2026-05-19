# user-service

> **Status:** 📋 Planned (Phase 3)  
> **Role:** User profile management

## Description

Manages user profile data (display name, avatar, preferences, timezone, locale). Deliberately separate from auth concerns — `auth-service` owns credentials, `user-service` owns the public-facing profile. A profile record is created automatically when the `auth.user.registered` event is received.

## Tech Stack

| Concern | Technology |
|---|---|
| Framework | FastAPI |
| Database | PostgreSQL (own instance) |
| Cache | Redis (profile cache, 5-min TTL) |
| File delegation | Calls `file-service` for avatar uploads |

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/users/me` | Own full profile |
| `PATCH` | `/users/me` | Update own profile |
| `DELETE` | `/users/me` | Soft-delete own account |
| `GET` | `/users/{id}` | Public profile (limited fields) |
| `POST` | `/users/me/avatar` | Upload avatar (delegates to file-service) |
| `GET` | `/users/me/preferences` | Notification & UI preferences |
| `PATCH` | `/users/me/preferences` | Update preferences |

## Events Consumed (Redis Streams)

| Stream | Action |
|---|---|
| `auth.user.registered` | Create profile record with default values |

## Events Published (Redis Streams)

| Stream | Trigger |
|---|---|
| `user.profile.updated` | Any profile field changed |
| `user.account.deleted` | Soft-delete initiated |

## Data Model (key fields)

```
users_profile
├── id            UUID PK (matches auth user_id)
├── display_name  TEXT
├── bio           TEXT nullable
├── avatar_url    TEXT nullable
├── timezone      TEXT default 'UTC'
├── locale        TEXT default 'en'
├── is_deleted    BOOL default false
├── deleted_at    TIMESTAMPTZ nullable
├── created_at    TIMESTAMPTZ
└── updated_at    TIMESTAMPTZ

user_preferences
├── user_id       UUID FK → users_profile.id
├── email_notifs  BOOL default true
├── inapp_notifs  BOOL default true
└── theme         TEXT default 'system'
```

## Security Notes

- Users may only modify their own profile (verified via `X-User-Id` header)
- Admin role can read all profiles
- Soft-delete only — hard delete requires separate admin action
- Avatar delegated to file-service (size + MIME validation enforced there)
