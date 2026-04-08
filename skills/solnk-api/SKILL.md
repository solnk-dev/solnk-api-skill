---
name: solnk-api
description: Publish content to 9 social platforms (X, Instagram, TikTok, YouTube, Facebook, LinkedIn, Pinterest, Threads, Bluesky) via the Solnk API. Use when the user wants to publish, schedule, or manage social media posts programmatically.
metadata:
  author: solnk
  version: "1.0.0"
---

# Solnk API

Solnk API lets you publish content to 9 social platforms with a single API call.

## Connection

```
Base URL: https://api.solnk.com/api/v1
Authorization: Bearer sk_live_xxxxxxxxxxxxxxxx
Content-Type: application/json
```

API keys: https://solnk.com/settings/api-keys

## What you can do

| Action | Method | Endpoint | Scope |
|--------|--------|----------|-------|
| List connected accounts | GET | /accounts | accounts:read |
| Get single account | GET | /accounts/{id} | accounts:read |
| Publish to platforms | POST | /publishes | posts:write |
| Get publish status | GET | /publishes/{id} | posts:read |
| List publishes | GET | /publishes | posts:read |
| Check usage & limits | GET | /usage | billing:read |
| Create media upload | POST | /media/uploads | media:write |
| Confirm media upload | POST | /media/uploads/{id}/confirm | media:write |
| List media | GET | /media | media:read |
| List webhooks | GET | /webhooks | webhooks:manage |
| Create webhook | POST | /webhooks | webhooks:manage |
| Delete webhook | DELETE | /webhooks/{id} | webhooks:manage |

## How to publish (step by step)

### 1. List accounts

```bash
curl https://api.solnk.com/api/v1/accounts?status=active \
  -H "Authorization: Bearer sk_live_your_key"
```

Note the `id` of each account you want to publish to.

### 2. Publish

```bash
curl -X POST https://api.solnk.com/api/v1/publishes \
  -H "Authorization: Bearer sk_live_your_key" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-key-123" \
  -d '{
    "content": "Hello from the Solnk API!",
    "targets": [
      { "account_id": "YOUR_X_ACCOUNT_ID" },
      { "account_id": "YOUR_LINKEDIN_ACCOUNT_ID" }
    ],
    "publish_mode": "immediate"
  }'
```

Each target can override `content` and `media_ids` for platform-specific text or media.

For scheduled posts, use `"publish_mode": "scheduled"` with `"scheduled_at": "2026-04-01T10:00:00Z"` (ISO 8601 UTC).

### 3. Check result

```bash
curl https://api.solnk.com/api/v1/publishes/PUBLISH_ID \
  -H "Authorization: Bearer sk_live_your_key"
```

Status: `queued` → `processing` → `success` / `partial_success` / `failed` / `cancelled`

## How to upload media

```bash
# Step 1: Get presigned upload URL
curl -X POST https://api.solnk.com/api/v1/media/uploads \
  -H "Authorization: Bearer sk_live_your_key" \
  -H "Content-Type: application/json" \
  -d '{"filename": "photo.jpg", "content_type": "image/jpeg", "size_bytes": 204800}'

# Step 2: Upload file to the returned upload_url (use the headers from step 1 response)
curl -X PUT "{upload_url}" -H "Content-Type: {headers.Content-Type}" --data-binary @photo.jpg

# Step 3: Confirm upload
curl -X POST https://api.solnk.com/api/v1/media/uploads/{media_id}/confirm \
  -H "Authorization: Bearer sk_live_your_key"
```

Then use the `media_id` in your publish request's `media_ids` array.

Supported types: `image/png`, `image/jpeg`, `image/webp`, `image/gif`, `video/mp4`, `video/quicktime`, `video/webm`

## How to check usage & limits

```bash
curl https://api.solnk.com/api/v1/usage \
  -H "Authorization: Bearer sk_live_your_key"
```

Returns `can_publish` (boolean), plan limits (`posts_per_month`, `social_accounts`, `media_storage_gb`), and current usage. A `null` limit means unlimited. Requires `billing:read` scope.

## How to manage webhooks

Get notified instead of polling for publish results.

```bash
# Subscribe to events
curl -X POST https://api.solnk.com/api/v1/webhooks \
  -H "Authorization: Bearer sk_live_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhook",
    "events": ["publish.request.completed", "publish.target.failed"]
  }'

# List existing webhooks
curl https://api.solnk.com/api/v1/webhooks \
  -H "Authorization: Bearer sk_live_your_key"

# Delete a webhook
curl -X DELETE https://api.solnk.com/api/v1/webhooks/{webhook_id} \
  -H "Authorization: Bearer sk_live_your_key"
```

Events: `publish.request.completed`, `publish.request.failed`, `publish.target.published`, `publish.target.failed`, `account.authorization.expired`

Webhooks use HMAC-SHA256 signatures. Retry policy: exponential backoff, max 6 retries over 12 hours.

## How to browse publish history

```bash
# List all publishes (paginated)
curl "https://api.solnk.com/api/v1/publishes?page=1&page_size=20" \
  -H "Authorization: Bearer sk_live_your_key"

# Filter by status, platform, account, or date
curl "https://api.solnk.com/api/v1/publishes?status=failed&platform=x&created_after=2026-04-01T00:00:00Z" \
  -H "Authorization: Bearer sk_live_your_key"
```

## Key rules

- **Idempotency**: Always send `Idempotency-Key` header on `POST /publishes` to prevent duplicates (cached 24h)
- **Rate limits**: `POST /publishes` = 30/min, all others = 120/min
- **Pagination**: Use `page` + `page_size` query params (not offset/limit)
- **IDs**: All UUIDs are v7 format
- **Timestamps**: ISO 8601 in UTC
- **Expired accounts**: If `status` is `expired`, the user must re-authorize in the Solnk dashboard before publishing

## When you need more detail

For complete request/response schemas, all parameters, error codes, and platform-specific settings:

**Fetch the full API reference:**
```
https://developers.solnk.com/openapi.json
```

Or browse the interactive docs: https://developers.solnk.com

## Error format

```json
{
  "error": {
    "type": "invalid_request",
    "code": "ACCOUNT_NOT_FOUND",
    "message": "Account not found",
    "request_id": "019d4200-..."
  }
}
```

Error types: `invalid_request` (400), `authentication_error` (401), `authorization_error` (403), `not_found` (404), `rate_limited` (429), `quota_exceeded` (429), `platform_error` (502), `internal_error` (500)
