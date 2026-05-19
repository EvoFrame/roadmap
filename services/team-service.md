# team-service

> **Status:** 📋 Planned (Phase 4)  
> **Role:** Teams, organizations, memberships, invitations

## Description

Manages multi-tenant team structures. A user can belong to multiple teams, each with a distinct role (`owner`, `admin`, `member`). Handles invitation flows via short-lived signed tokens sent by `notification-service`.

## Tech Stack

| Concern | Technology |
|---|---|
| Framework | FastAPI |
| Database | PostgreSQL (own instance) |
| Cache | Redis (invite token store, 48h TTL) |

## API Endpoints

| Method | Path | Description | Min role |
|---|---|---|---|
| `POST` | `/teams` | Create team | authenticated |
| `GET` | `/teams/{id}` | Team details | member |
| `PATCH` | `/teams/{id}` | Update team name/settings | admin |
| `DELETE` | `/teams/{id}` | Delete team | owner |
| `GET` | `/teams/{id}/members` | List members + roles | member |
| `POST` | `/teams/{id}/invites` | Send invite (email) | admin |
| `POST` | `/teams/invites/accept` | Accept invite token | authenticated |
| `DELETE` | `/teams/{id}/members/{user_id}` | Remove member | admin (or self) |
| `PATCH` | `/teams/{id}/members/{user_id}/role` | Change member role | owner |

## Events Consumed (Redis Streams)

| Stream | Action |
|---|---|
| `user.account.deleted` | Remove user from all teams; if sole owner, mark team as orphaned |

## Events Published (Redis Streams)

| Stream | Trigger |
|---|---|
| `team.member.invited` | Invite sent — carries email + invite token |
| `team.member.joined` | Invite accepted |
| `team.member.removed` | Member removed or left |
| `team.deleted` | Team deleted — triggers billing cancellation |

## Data Model (key fields)

```
teams
├── id          UUID PK
├── name        TEXT
├── slug        TEXT UNIQUE
├── owner_id    UUID (user_id of creator)
├── created_at  TIMESTAMPTZ
└── updated_at  TIMESTAMPTZ

team_members
├── team_id     UUID FK → teams.id
├── user_id     UUID
├── role        ENUM (owner, admin, member)
├── joined_at   TIMESTAMPTZ
└── PRIMARY KEY (team_id, user_id)
```

## Security Notes

- Invite tokens are RS256 JWTs with 48h expiry, signed with service key
- Only owners can delete teams or transfer ownership
- Invite acceptance validates token signature + expiry before creating membership
- One team must always have at least one owner (enforced at service layer)
