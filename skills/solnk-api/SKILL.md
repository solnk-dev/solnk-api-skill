---
name: solnk-api
description: Publish content to 9 social platforms (X, Instagram, TikTok, YouTube, Facebook, LinkedIn, Pinterest, Threads, Bluesky) via the Solnk API. Use when the user wants to publish, schedule, draft, or cross-post social media content; upload media; check connected accounts or plan usage; or pull post analytics (views, likes, comments, shares) — programmatically.
metadata:
  author: solnk
  version: "1.1.1"
---

# Solnk API

Solnk API lets you publish content to 9 social platforms with a single API call.

## Connection

```
Base URL: https://api.solnk.com/api/v1
Authorization: Bearer sk_xxxxxxxxxxxxxxxx
Content-Type: application/json
```

API keys: https://solnk.com/settings/api-keys

## What you can do

| Action | Method | Endpoint | Scope |
|--------|--------|----------|-------|
| List connected accounts | GET | /accounts | accounts:read |
| Get single account | GET | /accounts/{id} | accounts:read |
| Publish to platforms | POST | /publishes | posts:write |
| Confirm a draft publish | POST | /publishes/{id}/confirm | posts:write |
| Get publish status | GET | /publishes/{id} | posts:read |
| List publishes | GET | /publishes | posts:read |
| Cancel a draft/scheduled publish | DELETE | /publishes/{id} | posts:write |
| List post analytics | GET | /analytics/posts | analytics:read |
| Get post analytics (per-platform) | GET | /analytics/posts/{id} | analytics:read |
| Check usage & limits | GET | /usage | billing:read |
| Create media upload | POST | /media/uploads | media:write |
| Confirm media upload | POST | /media/uploads/{id}/confirm | media:write |
| List media | GET | /media | media:read |
| List webhooks | GET | /webhooks | webhooks:read |
| Create webhook | POST | /webhooks | webhooks:write |
| Delete webhook | DELETE | /webhooks/{id} | webhooks:write |

## How to publish (step by step)

### 1. List accounts

```bash
curl https://api.solnk.com/api/v1/accounts?status=active \
  -H "Authorization: Bearer sk_your_key"
```

Note the `id` of each account you want to publish to.

### 2. Publish

```bash
curl -X POST https://api.solnk.com/api/v1/publishes \
  -H "Authorization: Bearer sk_your_key" \
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

Each target can override `content`, `media_ids`, and `platform_settings` for platform-specific text, media, or options (see "Platform-specific settings" below).

`publish_mode` is one of:
- `"immediate"` — publish right away
- `"scheduled"` — publish later; also send `"scheduled_at": "2026-04-01T10:00:00Z"` (ISO 8601 UTC)
- `"draft"` — create but don't send; review it, then confirm (next step)

### 2b. Draft → review → confirm (optional, recommended for agents)

Create the post as a draft so a human can review the per-platform preview before anything goes live:

```bash
# Create a draft (nothing is published yet)
curl -X POST https://api.solnk.com/api/v1/publishes \
  -H "Authorization: Bearer sk_your_key" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: unique-key-123" \
  -d '{ "content": "Draft me", "publish_mode": "draft",
        "targets": [{ "account_id": "YOUR_X_ACCOUNT_ID" }] }'

# Confirm to publish now...
curl -X POST https://api.solnk.com/api/v1/publishes/PUBLISH_ID/confirm \
  -H "Authorization: Bearer sk_your_key" -H "Content-Type: application/json" -d '{}'

# ...or confirm with a schedule
curl -X POST https://api.solnk.com/api/v1/publishes/PUBLISH_ID/confirm \
  -H "Authorization: Bearer sk_your_key" -H "Content-Type: application/json" \
  -d '{ "scheduled_at": "2026-04-01T10:00:00Z" }'
```

Cancel a draft or not-yet-sent scheduled post (cannot cancel one already processing/published):

```bash
curl -X DELETE https://api.solnk.com/api/v1/publishes/PUBLISH_ID \
  -H "Authorization: Bearer sk_your_key"
```

### 3. Check result

```bash
curl https://api.solnk.com/api/v1/publishes/PUBLISH_ID \
  -H "Authorization: Bearer sk_your_key"
```

Status: `draft` → `queued` → `processing` → `success` / `partial_success` / `failed` / `cancelled`

The publish object returns the **aggregate** status only. To get each platform's live post URL and engagement, use the Analytics endpoints below.

## How to upload media

```bash
# Step 1: Get presigned upload URL
curl -X POST https://api.solnk.com/api/v1/media/uploads \
  -H "Authorization: Bearer sk_your_key" \
  -H "Content-Type: application/json" \
  -d '{"filename": "photo.jpg", "content_type": "image/jpeg", "size_bytes": 204800}'

# Step 2: Upload file to the returned upload_url (use the headers from step 1 response)
curl -X PUT "{upload_url}" -H "Content-Type: {headers.Content-Type}" --data-binary @photo.jpg

# Step 3: Confirm upload
curl -X POST https://api.solnk.com/api/v1/media/uploads/{media_id}/confirm \
  -H "Authorization: Bearer sk_your_key"
```

Then use the `media_id` in your publish request's `media_ids` array.

Supported types: `image/png`, `image/jpeg`, `image/webp`, `image/gif`, `video/mp4`, `video/quicktime`, `video/webm`

## How to check usage & limits

```bash
curl https://api.solnk.com/api/v1/usage \
  -H "Authorization: Bearer sk_your_key"
```

Returns `can_publish` (boolean), plan limits (`posts_per_month`, `social_accounts`, `media_storage_gb`), and current usage. A `null` limit means unlimited. Requires `billing:read` scope.

## How to manage webhooks

Get notified instead of polling for publish results.

```bash
# Subscribe to events
curl -X POST https://api.solnk.com/api/v1/webhooks \
  -H "Authorization: Bearer sk_your_key" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhook",
    "events": ["post.published", "post.failed"]
  }'

# List existing webhooks
curl https://api.solnk.com/api/v1/webhooks \
  -H "Authorization: Bearer sk_your_key"

# Delete a webhook
curl -X DELETE https://api.solnk.com/api/v1/webhooks/{webhook_id} \
  -H "Authorization: Bearer sk_your_key"
```

Events: `post.published`, `post.failed`, `post.pending_approval`, `post.approved`, `post.rejected`

Webhooks are scoped to the API key's team (team-scoped keys subscribe under their team; user-scoped keys use the caller's active team). HMAC-SHA256 signatures via `X-Solnk-Signature: sha256=<hex>` header. Retry policy: 10s timeout, exponential backoff 1m / 5m / 30m / 2h / 12h (max 6 attempts total).

## How to browse publish history

```bash
# List all publishes (paginated)
curl "https://api.solnk.com/api/v1/publishes?page=1&page_size=20" \
  -H "Authorization: Bearer sk_your_key"

# Filter by status, platform, account, or date
curl "https://api.solnk.com/api/v1/publishes?status=failed&platform=x&created_after=2026-04-01T00:00:00Z" \
  -H "Authorization: Bearer sk_your_key"
```

## How to read post analytics

Get views, likes, comments, shares, and the live post URLs for published posts. Requires `analytics:read` scope.

```bash
# List posts with rolled-up metrics across platforms
curl "https://api.solnk.com/api/v1/analytics/posts?page=1&page_size=20" \
  -H "Authorization: Bearer sk_your_key"

# Single post: per-platform breakdown, including each platform_post_url
curl https://api.solnk.com/api/v1/analytics/posts/POST_ID \
  -H "Authorization: Bearer sk_your_key"
```

List query params: `page`, `page_size`, `platform`, `sort`, `order`.

Each post returns `metrics` (`total_views`, `total_likes`, `total_comments`, `total_shares`, `last_sync_at`). The single-post response adds a `platforms[]` array, each with `platform`, `platform_post_url` (the live link), per-platform `metrics`, and `platform_specific_data` (e.g. impressions, reach, saves). Metrics sync periodically — `last_sync_at` may be `null` right after publishing.

## Platform-specific settings

Set per target via `platform_settings`. It's a **flat object** — the keys below go directly inside it (do **not** nest under a platform name). Each target already knows its platform from the account. Only the keys listed here are used; anything else is ignored.

```jsonc
"targets": [
  { "account_id": "YOUR_YT_ID",  "platform_settings": { "title": "Launch demo", "privacyStatus": "unlisted", "tags": ["solnk"], "madeForKids": false } },
  { "account_id": "YOUR_PIN_ID", "platform_settings": { "boardId": "987654321", "title": "Solnk launch", "link": "https://solnk.com" } }
]
```

- **x** — `reply_to`, `quote_tweet`, `sensitive` (bool), `for_super_followers_only` (bool)
- **linkedin** — `title`, `visibility` (`PUBLIC` | `CONNECTIONS` | `LOGGED_IN_MEMBERS`), `target_audience` (object)
- **facebook** — `postType` (`post` | `reels`)
- **instagram** — `first_comment` (auto-posted as the first comment)
- **threads** — `link_attachment` (url), `topic_tag`
- **bluesky** — `first_comment`, `reply` (`{root,parent}`), `embed` (`{external:{uri,title,description}}`)
- **pinterest** — `boardId`, `title`, `link`, `note`, `alt_text`, `cover_image_key_frame_time` (number). Multi-account: `per_account: { "<account_id>": { "boardId": "..." } }`
- **youtube** — `title` (required for video), `privacyStatus` (`private` | `public` | `unlisted`), `category_id`, `default_language`, `default_audio_language`, `tags` (array), `madeForKids` (bool)
- **tiktok** — `tiktok_post_mode` (`DIRECT_POST` | `MEDIA_UPLOAD`), `title` (photo posts, ≤90 chars), `privacy_level` (`PUBLIC_TO_EVERYONE` | `MUTUAL_FOLLOW_FRIENDS` | `FOLLOWER_OF_CREATOR` | `SELF_ONLY`), `allows_comments`, `allows_duet`, `allows_stitch`, `brand_content_toggle`, `brand_organic_toggle`, `auto_add_music` (all bool)

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

Error types: `invalid_request` (400), `authentication_error` (401), `authorization_error` (403), `not_found` (404), `conflict` (409, e.g. reused idempotency key with a different body), `rate_limited` (429), `quota_exceeded` (429), `platform_error` (502), `internal_error` (500)
