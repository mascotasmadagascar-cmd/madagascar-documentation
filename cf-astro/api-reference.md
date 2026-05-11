# cf-astro — API Reference

All REST API endpoints exposed by the cf-astro public website Worker.

---

## Common Patterns

### CSRF Protection

All mutating POST endpoints call `assertOrigin()` from `src/lib/security.ts`. Allowed origins:

- `https://madagascarhotelags.com`
- `https://www.madagascarhotelags.com`
- `https://cf-astro.pages.dev`
- `http://localhost:*` (any port)

Requests from other origins receive `403 Forbidden`.

### Rate Limiting

Rate limiting uses Upstash Redis sliding window with an in-memory `Map` fallback. On Redis error, the fallback is used. If both fail, the endpoint returns `503` (fail-closed — no bypass on error).

Response headers on rate-limited responses (`429`):
```
X-RateLimit-Limit: {limit}
X-RateLimit-Remaining: 0
X-RateLimit-Reset: {unix timestamp}
Retry-After: {seconds}
```

### IP Extraction

Client IP is extracted in priority order:
1. `cf-connecting-ip` header (Cloudflare spoofproof)
2. `x-forwarded-for` header first value
3. UUID fallback (if neither header present)

### Timing-Safe Comparisons

All secret/token verification uses `timingSafeEq()` from `src/lib/security.ts` to prevent timing oracle attacks.

---

## Endpoints

---

### POST `/api/booking`

Submit a new pet hotel booking.

**Auth**: `assertOrigin()` CSRF check
**Rate limit**: 5 requests / 60 seconds / IP

**Request body** (JSON, Zod-validated):

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `owner_name` | string | Yes | |
| `owner_email` | string | Yes | |
| `owner_phone` | string | Yes | |
| `emergency_contact` | string | No | |
| `special_instructions` | string | No | |
| `service` | string | Yes | `hotel`, `daycare`, or `relocation` |
| `check_in_date` | string (date) | Yes | ISO date `YYYY-MM-DD` |
| `check_out_date` | string (date) | Yes | ISO date `YYYY-MM-DD` |
| `transport_type` | string | Yes | `none`, `pickup`, `dropoff`, or `both` |
| `transport_address` | string | No | Required if transport_type != `none` |
| `pets` | array | Yes | At least one pet required |
| `pets[].pet_name` | string | Yes | |
| `pets[].pet_type` | string | Yes | |
| `pets[].pet_breed` | string | Yes | |
| `pets[].pet_age` | string | Yes | |
| `pets[].pet_weight` | string | No | |
| `consent.consentType` | string | Yes | `booking` |
| `consent.version` | string | Yes | |
| `consent.hash` | string | Yes | 64-char hex SHA-256 of consent text |
| `consent.mechanism` | string | Yes | `checkbox_click` or `button_click` |
| `consent.locale` | string | Yes | `es` or `en` |
| `consent.fingerprint` | object | No | Browser fingerprint data |
| `posthog_session_id` | string | No | For funnel analytics |

**Processing — 3-phase insert**:

- Phase 1 (blocking): INSERT `consent_records` → INSERT `bookings` (sequential, transactional)
- Phase 2 (waitUntil, non-blocking): INSERT `booking_pets[]` + INSERT `email_audit_logs` (parallel)
- Phase 3 (waitUntil): Push to `EMAIL_QUEUE` (admin notification + customer confirmation), write Analytics Engine datapoint

**Response (200)**:
```json
{
  "success": true,
  "bookingRef": "MAD-XXXXXXXXXX",
  "consentId": "uuid-v4",
  "emailsQueued": true,
  "degraded": false,
  "whatsappUrl": "https://wa.me/52..."
}
```

`degraded: true` means Phase 1 succeeded (booking saved) but Phase 2 or 3 failed (email may be missing). Sentry captures the error.

**Error responses**:

| Code | Condition |
|------|-----------|
| 400 | Zod validation failed — body in response |
| 403 | Origin not allowed (CSRF) |
| 429 | Rate limit exceeded |
| 500 | Phase 1 DB insert failed |

---

### POST `/api/contact`

Submit a contact form inquiry.

**Auth**: `assertOrigin()` CSRF check
**Rate limit**: 3 requests / 60 seconds / IP

**Request body** (JSON):

| Field | Type | Required |
|-------|------|----------|
| `name` | string | Yes |
| `email` | string | Yes |
| `phone` | string | No |
| `service` | string | No |
| `message` | string | Yes |
| `locale` | string | No | Defaults to `es` |

**Processing**:
- Phase 1 (blocking): INSERT into `contact_messages` table
- Phase 2 (waitUntil): INSERT `email_audit_logs` + push to `EMAIL_QUEUE`

**Response (200)**:
```json
{
  "success": true,
  "ref": "uuid-v4-tracking-id"
}
```

**Error responses**:

| Code | Condition |
|------|-----------|
| 400 | Missing required fields |
| 403 | Origin not allowed |
| 429 | Rate limit exceeded |
| 500 | DB insert failed |

---

### POST `/api/consent`

Record a consent event (cookie banner or booking consent).

**Auth**: None (public)
**Rate limit**: 5 requests / 60 seconds / IP

**Request body** (JSON):

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `consentType` | string | Yes | `cookies_essential`, `cookies_analytics`, or `booking` |
| `version` | string | Yes | Consent text version |
| `hash` | string | Yes | 64-char hex SHA-256 of the consent text displayed |
| `mechanism` | string | Yes | `checkbox_click` or `button_click` |
| `locale` | string | Yes | `es` or `en` |
| `fingerprint` | object | No | Browser fingerprint data |

**Validation**: `hash` must be exactly 64 hex characters (SHA-256 of consent text).

**Side effects**: Captures geolocation from CF headers (`cf-ipcountry`, region, city), IP, user-agent, interaction proof. For cookie consents, email is stored as anonymized placeholder: `Visitor {uuid.slice(0,8)}`.

**Response (200)**:
```json
{
  "success": true,
  "consentId": "uuid-v4"
}
```

**Error responses**:

| Code | Condition |
|------|-----------|
| 400 | Invalid hash format or missing fields |
| 429 | Rate limit exceeded |
| 500 | DB insert failed |

---

### POST `/api/privacy/arco`

Submit an ARCO (Acceso, Rectificación, Cancelación, Oposición) data privacy rights request.

**Auth**: `assertOrigin()` CSRF check
**Rate limit**: 3 requests / 60 seconds / IP

**Request body** (JSON):

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `requestType` | string | Yes | `access`, `rectification`, `cancellation`, or `opposition` |
| `requesterName` | string | Yes | |
| `requesterEmail` | string | Yes | |
| `description` | string | Yes | |

**Side effects**: INSERT into `privacy_requests` table (Supabase). Email notification queued to admin (non-fatal if queue fails).

**Response (200)**:
```json
{
  "success": true,
  "requestId": "uuid-v4",
  "message": "Your request has been submitted..."
}
```

---

### POST `/api/arco/submit`

Upload identity document as part of an ARCO legal request. Accepts `multipart/form-data`.

**Auth**: `assertOrigin()` CSRF check + Cloudflare Turnstile CAPTCHA
**Rate limit**: 3 requests / 60 seconds / IP

**6-layer file validation** (in order):

1. Rate limit check (Upstash Redis)
2. Cloudflare Turnstile CAPTCHA verification
3. File size must be ≤ 5 MB
4. File extension must be in whitelist: `jpg`, `jpeg`, `png`, `webp`
5. Magic bytes validation — read binary file header, verify matches expected image format (<1ms CPU)
6. MIME type cross-check against extension

**Request** (multipart/form-data):

| Field | Type | Notes |
|-------|------|-------|
| `turnstile_token` | string | From Turnstile widget |
| `arco_right` | string | `access`, `rectification`, `cancellation`, or `opposition` |
| `full_name` | string | |
| `email` | string | |
| `phone` | string | Optional |
| `description` | string | |
| `identity_document` | File | Image file — validated through 6 layers |

**Storage**: R2 path `arco/{year}/{ticketNumber}/{uuid}` in `ARCO_DOCS` bucket.

**D1 record**: INSERT into `legalRequests` table in `madagascar-db`.

**Response (200)**:
```json
{
  "success": true,
  "ticketNumber": "ARCO-123456",
  "documentUrl": "r2-internal-path",
  "expiresAt": "ISO-timestamp"
}
```

**Error responses**:

| Code | Condition |
|------|-----------|
| 400 | File validation failed (size, extension, magic bytes, MIME) |
| 400 | Turnstile verification failed |
| 403 | Origin not allowed |
| 429 | Rate limit exceeded |
| 500 | R2 upload or D1 insert failed |

---

### GET `/api/arco/get-document`

Download an identity document from R2 (admin use only).

**Auth**: `X-Admin-Secret: {ARCO_ADMIN_SECRET}` header (timing-safe comparison)
**Rate limit**: None

**Query parameters**:

| Parameter | Type | Required |
|-----------|------|----------|
| `ticket` | string | Yes | ARCO ticket number e.g. `ARCO-123456` |

**Response** (200): File stream with headers:
```
Content-Disposition: attachment; filename="{filename}"
X-Ticket: ARCO-123456
X-Requester: requester@email.com
X-ARCO-Right: access
```

**Error responses**:

| Code | Condition |
|------|-----------|
| 401 | Missing or invalid `X-Admin-Secret` |
| 404 | Ticket not found in D1 |
| 404 | Document not found in R2 |

---

### GET `/api/health`

Health probe for uptime monitoring systems.

**Auth**: `Authorization: Bearer {HEALTH_CHECK_SECRET}` (timing-safe)
**Rate limit**: None

**Checks performed**:
1. D1 latency — executes `SELECT 1` and measures round-trip
2. ISR_CACHE KV latency — executes a `get` on a test key and measures round-trip

**Response (200 — ok)**:
```json
{
  "status": "ok",
  "timestamp": "2026-05-10T12:00:00.000Z",
  "checks": {
    "d1": { "ok": true, "latencyMs": 12 },
    "kv": { "ok": true, "latencyMs": 4 }
  }
}
```

**Response (207 — degraded)**: Same shape, `status: "degraded"`, one or both checks have `ok: false`.

**Error responses**:

| Code | Condition |
|------|-----------|
| 401 | Missing or invalid Authorization header |

---

### POST `/api/revalidate`

Purge ISR cache for specific paths and/or write CMS content blocks to KV. Called by cf-admin on CMS updates.

**Auth**: `Authorization: Bearer {REVALIDATION_SECRET}` (timing-safe). Secret must match the value set in cf-admin.
**Rate limit**: None

**Request body** (JSON):

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `paths` | string[] | Yes | URL paths to invalidate e.g. `["/", "/en/services"]` |
| `cmsData` | object | No | Key/value pairs to write to KV; keys must be in allowlist |

**CMS key allowlist**: `hero`, `services`, `pricing`, `gallery`, `testimonials`, `faqs`, `faq_draft`, `about`, `contact`, `franchise`, `blog_index`, and any key matching `seo_*` or `blog_draft_*`.

**Side effects**:
1. For each path: deletes `isr:{normalizedPath}#{buildId}` from `ISR_CACHE` KV
2. For each `cmsData` key in allowlist: writes JSON to KV
3. Pings Bing IndexNow via `src/lib/indexnow.ts`
4. Pings Yandex IndexNow via `src/lib/indexnow.ts`

**Response (200)**:
```json
{
  "revalidated": true,
  "message": "Cache purged and CMS data written",
  "paths": ["/", "/en/services"],
  "cmsKeysWritten": ["hero", "services"],
  "now": "2026-05-10T12:00:00.000Z"
}
```

---

### POST `/api/analytics/track`

Track a client-side CTA or funnel event.

**Auth**: None (public)
**Rate limit**: 60 requests / 60 seconds / IP

**Supported events**:

| Event name | Trigger |
|-----------|---------|
| `whatsapp_click` | WhatsApp button clicked |
| `phone_click` | Phone number clicked |
| `email_click` | Email address clicked |
| `booking_cta` | Booking CTA button clicked |
| `booking_funnel_step1` | Step 1 of booking wizard entered |
| `booking_submit` | Booking form submitted |
| `faq_expand` | FAQ item expanded |
| `nav_click` | Navigation link clicked |

**Request body** (JSON):

| Field | Type | Required |
|-------|------|----------|
| `event` | string | Yes | Must be one of the supported events above |
| `page` | string | No | Current page URL |

**Side effects**: Writes to Analytics Engine via `env.ANALYTICS.writeDataPoint()` (non-blocking).

**Response**: `HTTP 204 No Content`

---

### GET `/api/analytics/summary`

Get a KV-cached weekly analytics digest.

**Auth**: `Authorization: Bearer {REVALIDATION_SECRET}`
**Rate limit**: None

**Query parameters**:

| Parameter | Type | Default | Constraints |
|-----------|------|---------|-------------|
| `days` | integer | `7` | 1–30 |

**Response (200)**:
```json
{
  "period": { "days": 7, "from": "...", "to": "..." },
  "pageViews": 0,
  "topPages": [],
  "topCountries": [],
  "bookingEvents": 0
}
```

---

### POST `/api/admin/generate-blog-draft`

Generate an 800-word blog post using Workers AI.

**Auth**: `Authorization: Bearer {ADMIN_AI_SECRET}` (timing-safe)
**Rate limit**: None (quota-protected by Workers AI 10K neurons/day)

**Request body** (JSON):

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `title` | string | Yes | Blog post title |
| `keywords` | string[] | No | SEO keywords to incorporate |
| `lang` | string | No | `es` or `en`; defaults to `es` |

**Processing**:
1. Calls `env.AI.run('@cf/meta/llama-3-8b-instruct', { prompt })` with blog context
2. Generates 800-word HTML-structured blog post
3. Stores in D1 `cms_content` table (`block_type: 'blog_draft'`)

**Response (200)**:
```json
{
  "success": true,
  "draft": {
    "title": "...",
    "slug": "...",
    "excerpt": "...",
    "content": "<article>...</article>",
    "lang": "es"
  }
}
```

---

### POST `/api/admin/generate-faqs`

Generate 12 FAQ question/answer pairs using Workers AI.

**Auth**: `Authorization: Bearer {ADMIN_AI_SECRET}` (timing-safe)
**Rate limit**: None

**Request body**: None required.

**Processing**:
1. Calls Workers AI with prompt for 12 FAQ Q&As in Spanish (Mexico pet hotel context)
2. Stores result in D1: `block_type='seo'`, `block_key='faq_draft'`

**Response (200)**:
```json
{
  "success": true,
  "count": 12,
  "faqs": [
    { "question": "...", "answer": "..." }
  ]
}
```

---

### POST `/api/webhooks/resend`

Receive email delivery lifecycle events from Resend via Svix webhooks.

**Auth**: Svix HMAC-SHA256 verification using headers `svix-id`, `svix-timestamp`, `svix-signature`
**Note**: Svix npm package is incompatible with Workers; signature is verified manually.
**Replay protection**: 5-minute timestamp tolerance (reject events older than 5 minutes)

**Events handled**:

| Resend event | Status written to DB |
|-------------|---------------------|
| `email.sent` | `sent_to_provider` |
| `email.delivered` | `delivered` |
| `email.bounced` | `bounced` |
| `email.opened` | `opened` |
| `email.clicked` | `clicked` |
| `email.complained` | `complained` |
| `email.delivery_delayed` | `delayed` |

**Side effects**: Updates `email_audit_logs` table — sets `status` and appends event to `delivery_events` JSONB array. Idempotent via `svix_id` (duplicate webhooks are ignored).

**Response**: Always `HTTP 200 OK` (even on errors) to prevent Resend from retrying on 5xx.

---

### GET `/api/media/[...path]`

Image proxy that streams from R2 `IMAGES` bucket. Dev environment only — in production, images are served directly from CDN.

**Auth**: None
**Rate limit**: None

**Behavior**:
- Path maps to R2 object key
- Content-Type guessed from file extension
- Response header: `Cache-Control: public, max-age=31536000, immutable`

---

### GET+POST `/api/ingest/[...path]` (PostHog Proxy)

Privacy-preserving proxy for PostHog analytics. All client-side PostHog calls go through this Worker instead of directly to PostHog servers.

**Auth**: None
**Rate limit**: None

**Behavior**:
- Forwards all methods (GET, POST) to `https://us.i.posthog.com` with the same path and body
- Forwarded request headers: `content-type`, `accept`, `user-agent`
- Stripped from proxy response: `set-cookie`, `x-frame-options`

---

### POST `/api/mcp`

JSON-RPC 2.0 MCP (Model Context Protocol) server for Claude and other AI integrations.

**Auth**: None
**Rate limit**: None

**Supported JSON-RPC methods**:

| Method | Description |
|--------|-------------|
| `initialize` | MCP handshake |
| `tools/list` | List available tools |
| `tools/call` | Call a tool by name |

**Available tools**:

| Tool name | Returns |
|-----------|---------|
| `get_services` | Services list with pricing |
| `get_locations` | Hotel location(s) with hours |
| `get_booking_url` | Booking page URL |
| `get_faq` | FAQ content |

**Request format** (JSON-RPC 2.0):
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "get_services",
    "arguments": {}
  },
  "id": 1
}
```

---

### GET `/api/test-services`

Smoke test that verifies Analytics Engine binding is functional.

**Auth**: `Authorization: Bearer {HEALTH_CHECK_SECRET}` (timing-safe)
**Rate limit**: None

**Side effects**: Writes one test event to Analytics Engine.

**Response (200)**:
```json
{
  "success": true,
  "message": "Analytics Engine write test passed",
  "timestamp": "2026-05-10T12:00:00.000Z"
}
```

---

## Rate Limit Summary

| Endpoint | Limit | Window |
|----------|-------|--------|
| `POST /api/booking` | 5 | 60s |
| `POST /api/consent` | 5 | 60s |
| `POST /api/contact` | 3 | 60s |
| `POST /api/privacy/arco` | 3 | 60s |
| `POST /api/arco/submit` | 3 | 60s |
| `POST /api/analytics/track` | 60 | 60s |
