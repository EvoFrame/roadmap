# file-service

> **Status:** 📋 Planned (Phase 7)  
> **Role:** File upload, storage, and secure download

## Description

Handles multipart file uploads, stores files to local filesystem (dev) or S3/MinIO (prod), and generates short-lived presigned URLs for secure direct downloads. Tracks file ownership, size, and MIME type. Files are private by default.

## Tech Stack

| Concern | Technology |
|---|---|
| Framework | FastAPI |
| Database | PostgreSQL (own instance) |
| Cache | Redis (presigned URL metadata, short TTL) |
| Dev storage | Local filesystem |
| Prod storage | S3-compatible via `boto3` (AWS S3 or MinIO) |

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/files/upload` | Multipart upload, returns file ID |
| `GET` | `/files/{id}` | File metadata (name, size, MIME, owner) |
| `GET` | `/files/{id}/download` | Presigned download URL (expires in 15min) |
| `DELETE` | `/files/{id}` | Delete file (owner or admin only) |
| `GET` | `/files` | List own files (paginated) |

## Storage Strategy

```
dev  → LOCAL_STORAGE_PATH/{user_id}/{file_id}/{original_name}
prod → s3://{BUCKET}/{user_id}/{file_id}/{original_name}
```

Switching between backends is controlled by the `STORAGE_BACKEND` env var (`local` | `s3`).

## Events Consumed (Redis Streams)

| Stream | Action |
|---|---|
| `user.account.deleted` | Soft-delete all files owned by user; schedule physical deletion |

## Events Published (Redis Streams)

| Stream | Trigger |
|---|---|
| `file.uploaded` | Upload successfully persisted |
| `file.deleted` | File removed from storage |

## Data Model (key fields)

```
files
├── id              UUID PK
├── owner_id        UUID (user_id)
├── original_name   TEXT
├── storage_key     TEXT (path or S3 key)
├── mime_type       TEXT
├── size_bytes      BIGINT
├── is_deleted      BOOL default false
├── deleted_at      TIMESTAMPTZ nullable
├── created_at      TIMESTAMPTZ
└── updated_at      TIMESTAMPTZ
```

## Security Notes

- Max upload size enforced at service layer (default 50 MB, configurable)
- MIME type validated against file content (magic bytes), not just the `Content-Type` header
- Filenames sanitized (path traversal prevention)
- Presigned URLs expire in 15 minutes (configurable)
- ClamAV or external virus scan hook on upload (optional, async)
- Users can only access or delete their own files
- `boto3` credentials injected via environment, never hardcoded
