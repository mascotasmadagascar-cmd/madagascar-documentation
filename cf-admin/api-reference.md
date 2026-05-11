# cf-admin — API Reference

All 45 API endpoints. Every endpoint (except those marked `—`) requires a valid `__Host-admin_session` cookie. The middleware validates the session and PLAC access before the route handler runs.

---

## Auth Mechanism

All protected endpoints:
- Read `__Host-admin_session` cookie
- Validate KV session exists and is not expired / revoked
- Check PLAC `accessMap` for the requested path
- Check RBAC `hasPermission(session.role, requiredRole)` inside the handler

Role column below shows the **minimum role** required. Higher-privilege roles inherit access automatically.

---

## 1. Authentication

| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/auth/logout` | any | Destroy KV session, clear cookie, redirect to CF Access logout URL |
| POST | `/api/auth/logout` | any | Same as GET — supports both methods |
| POST | `/api/auth/dev-login` | — | Local dev only; auto-bootstrap session from `LOCAL_DEV_ADMIN_EMAIL`; 404 in production |

### `POST /api/auth/dev-login`

Only available when `SITE_URL` contains `localhost`. Returns 404 in production.

**Response (200):**
```json
{ "success": true, "redirect": "/dashboard" }
```

Side effect: Sets `admin_session` cookie (local dev name).

---

## 2. Users & Access Management

| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/users/index` | admin | List authorized users — paginated, filterable by role / is_active |
| POST | `/api/users/manage` | owner | Create new admin user (whitelist insert + optional page overrides) |
| PATCH | `/api/users/manage` | owner | Update user (role, is_active, display_name) |
| DELETE | `/api/users/manage` | owner | Remove user — triggers 3-layer force-kick + D1 delete |
| POST | `/api/users/access` | super_admin | PLAC provisioning: grant / revoke / reset a page override for a user |
| GET | `/api/users/access-data` | super_admin | Fetch a user's current full PLAC access map |
| GET | `/api/users/activity` | admin | Activity list for a user |
| POST | `/api/users/pages` | owner | Set page registry entries for a user |
| POST | `/api/users/probes` | super_admin | Simulate access check for a given user on a given path |
| GET | `/api/users/[id]/login-history` | owner | Login forensics log for specific user (from `admin_login_logs`) |
| GET | `/api/users/[id]/session-status` | owner | Check whether user has any active KV sessions |
| POST | `/api/users/force-kick` | owner | Force logout user — all sessions, 3-layer cascade |
| GET | `/api/users/cf-access-audit` | owner | CF Access audit trail for a user |

### `GET /api/users/index`

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | number | 1 | Page number |
| `limit` | number | 25 | Records per page |
| `role` | string | — | Filter by role |
| `is_active` | 0 \| 1 | — | Filter by active status |

**Response (200):**
```json
{
  "data": [
    {
      "id": "uuid",
      "email": "admin@example.com",
      "role": "admin",
      "display_name": "Admin User",
      "is_active": 1,
      "is_hidden": 0,
      "created_at": "2026-01-01T00:00:00Z"
    }
  ],
  "total": 12,
  "page": 1
}
```

Note: Users with `is_hidden = 1` are excluded from results unless the requesting user has higher privilege than the hidden user's role.

### `POST /api/users/manage`

Rate limited: 10 requests per hour (Upstash Redis).

**Request body:**
```json
{
  "email": "newstaff@example.com",
  "role": "staff",
  "display_name": "New Staff Member",
  "page_overrides": [
    { "page_path": "/dashboard/bookings", "granted": 1 }
  ]
}
```

### `PATCH /api/users/manage`

Rate limited: 10 requests per hour.

**Request body:**
```json
{
  "id": "uuid",
  "role": "admin",
  "is_active": 1,
  "display_name": "Updated Name"
}
```

Side effect: If role or is_active changed, target user's sessions are revoked via KV reverse-index lookup.

### `POST /api/users/access`

Rate limited: 5 requests per minute (PLAC provisioning — highest sensitivity).

**Request body:**
```json
{
  "user_id": "uuid",
  "page_path": "/dashboard/users",
  "action": "grant" | "revoke" | "reset",
  "reason": "Temporary access for audit week"
}
```

`reset` removes the override row, reverting to role-default behavior.

### `POST /api/users/probes`

Simulates what `checkPageAccess()` would return for a given user and path, without affecting any state.

**Request body:**
```json
{
  "user_id": "uuid",
  "path": "/dashboard/content/gallery"
}
```

**Response (200):**
```json
{
  "allowed": true,
  "reason": "role_default" | "explicit_grant" | "explicit_deny",
  "required_role": "admin",
  "user_role": "admin"
}
```

### `POST /api/users/force-kick`

**Request body:**
```json
{ "user_id": "uuid" }
```

Executes all 3 force-logout layers. Audit-logs the action.

---

## 3. Audit & Logging

| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/audit/logs` | admin | Query `admin_audit_log` (filters: action, module, actor, date range) |
| DELETE | `/api/audit/logs` | dev | Bulk delete audit logs — 100 records per batch |
| GET | `/api/audit/login-logs` | owner | Query `admin_login_logs` — full forensics data including CF fields |
| GET | `/api/audit/consent` | admin | Consent audit trail |
| GET | `/api/audit/emails` | admin | Email delivery logs |
| GET | `/api/audit/receipts` | admin | Transaction receipts |
| GET | `/api/audit/sessions` | owner | Session audit records |
| POST | `/api/audit/prune` | dev | Delete old audit logs per retention policy |
| POST | `/api/audit/silence` | dev | Toggle `auditSilenced` flag in current session |
| GET | `/api/audit/stats` | admin | Audit log statistics (counts by module, action, actor) |

### `GET /api/audit/logs`

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `action` | string | Filter by action enum (`login`, `update`, `delete`, etc.) |
| `module` | string | Filter by module enum (`users`, `content`, `bookings`, etc.) |
| `actor` | string | Filter by `user_email` |
| `from` | ISO date | Start of date range |
| `to` | ISO date | End of date range |
| `limit` | number | Default 50 |
| `offset` | number | Default 0 |

**Response (200):**
```json
{
  "data": [
    {
      "id": "hex16",
      "user_id": "uuid",
      "user_email": "admin@example.com",
      "user_role": "admin",
      "action": "update",
      "module": "content",
      "target_id": "hero_title_es",
      "target_type": "cms_block",
      "details": { "field": "content", "prev": "Old title" },
      "ip_hash": "a3f9...",
      "created_at": "2026-05-10T12:00:00Z"
    }
  ],
  "total": 1432
}
```

### `POST /api/audit/silence`

DEV-only. Toggles `session.auditSilenced = true/false` in KV. When silenced, all `auditLog()` calls are no-ops. Expires with the session (24-hour max).

**Request body:**
```json
{ "silenced": true }
```

---

## 4. Diagnostics & Health

| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/health` | — | Public health check: D1, R2, KV, Supabase latency |
| POST | `/api/diagnostics/run` | dev | Execute full diagnostic test suite (results persisted via waitUntil) |
| GET | `/api/diagnostics/results` | dev | Fetch historical diagnostic run results from `admin_diagnostic_runs` |
| GET | `/api/diagnostics/infrastructure` | admin | Real-time infrastructure status |
| GET | `/api/diagnostics/ping` | — | Simple liveness ping — returns `{ "ok": true }` |

### `GET /api/health`

No auth required. Returns latency for each binding.

**Response (200):**
```json
{
  "status": "healthy" | "degraded" | "down",
  "timestamp": "2026-05-10T12:00:00Z",
  "components": {
    "d1":       { "status": "up", "latencyMs": 2 },
    "kv":       { "status": "up", "latencyMs": 4 },
    "r2":       { "status": "up", "latencyMs": 8 },
    "supabase": { "status": "up", "latencyMs": 35 }
  }
}
```

### `POST /api/diagnostics/run`

Executes all diagnostic test categories. Results are written to `admin_diagnostic_runs` via `ctx.waitUntil()`. Returns immediately with a run ID.

**Response (200):**
```json
{
  "run_id": "uuid",
  "status": "started",
  "message": "Diagnostic suite running in background"
}
```

---

## 5. Content & CMS

| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| POST | `/api/content/services` | admin | Update services CMS blocks |
| POST | `/api/content/reviews` | admin | Update reviews blocks |
| POST | `/api/content/blocks` | admin | Generic CMS block upsert |
| POST | `/api/media/upload` | admin | Upload image to R2 + D1 upsert + CDN URL returned |
| GET | `/api/media/gallery` | admin | Fetch gallery images list (ETag cached) |
| POST | `/api/media/revalidate` | admin | Trigger ISR revalidation on cf-astro for given paths |

### `POST /api/content/blocks`

**Request body:**
```json
{
  "id": "hero_title_es",
  "page": "hero",
  "type": "text" | "image_url" | "json",
  "content": "Bienvenidos a Madagascar"
}
```

Side effect: Triggers ISR revalidation for affected pages via ASTRO_SERVICE binding.

### `POST /api/media/upload`

**Request**: `multipart/form-data` with `file` field.

Validates: JPEG / PNG / WebP only, file size < 5MB.

**Response (200):**
```json
{
  "success": true,
  "url": "https://cdn.madagascarhotelags.com/{uuid}.jpg",
  "key": "{uuid}.jpg"
}
```

### `POST /api/media/revalidate`

**Request body:**
```json
{
  "paths": ["/", "/es/", "/en/services"]
}
```

Calls cf-astro via ASTRO_SERVICE binding with `REVALIDATION_SECRET` header. Retries 3 times with exponential backoff (300ms → 600ms → 900ms).

---

## 6. Bookings

| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/bookings/index` | admin | List bookings — Supabase query merged with D1 shadow state |
| GET | `/api/bookings/[id]` | admin | Fetch single booking detail (Supabase + D1 shadow state) |
| POST | `/api/bookings/[id]/state` | admin | Update booking operational status in D1 shadow state |

### `GET /api/bookings/index`

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter by `operational_status` |
| `limit` | number | Default 50 |
| `offset` | number | Default 0 |

**Response (200):**
```json
{
  "data": [
    {
      "id": "uuid",
      "guest_name": "María García",
      "email": "maria@example.com",
      "check_in": "2026-06-01",
      "check_out": "2026-06-05",
      "operational_status": "confirmed",
      "is_deleted": 0,
      "internal_notes": null
    }
  ],
  "total": 85
}
```

### `POST /api/bookings/[id]/state`

Updates D1 `admin_booking_state` (shadow state). Does not modify Supabase.

**Request body:**
```json
{
  "operational_status": "confirmed" | "in_progress" | "completed" | "cancelled",
  "internal_notes": "VIP guest — extra care",
  "is_deleted": 0
}
```

---

## 7. Settings

| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/settings/portal` | owner | Get portal-wide settings (CMS config, branding) |
| POST | `/api/settings/portal` | owner | Update portal settings |
| GET | `/api/settings/user` | staff | Get per-user settings for current session user |
| POST | `/api/settings/user` | staff | Update per-user settings (theme, locale, preferences) |

---

## 8. Features & Pages

| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| POST | `/api/features/toggle` | dev | Enable or disable a feature flag in `admin_feature_flags` |
| GET | `/api/pages/toggle` | super_admin | Get page registry status (all `admin_pages` with `is_active`) |
| POST | `/api/pages/toggle` | super_admin | Toggle `is_active` for a page in the registry |

### `POST /api/features/toggle`

**Request body:**
```json
{
  "flag_key": "new_booking_ui",
  "is_enabled": true
}
```

Side effect: Triggers ISR revalidation on cf-astro for affected pages.

---

## 9. Dashboard

| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/dashboard/metrics` | staff | Dashboard KPI metrics (booking counts, revenue summary, active users) |

---

## 10. Chatbot Proxy

| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| POST | `/api/chatbot/[...path]` | admin | Wildcard proxy to cf-chatbot via CHATBOT_SERVICE service binding |

All requests to `/api/chatbot/*` are forwarded to the cf-chatbot worker via `CHATBOT_SERVICE` binding with `X-Admin-Key: {CHATBOT_ADMIN_API_KEY}` appended. The catch-all `[...path]` parameter forwards the sub-path directly.

---

## 11. System

| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| GET | `/api/system/pages` | super_admin | List all rows from `admin_pages` D1 table |
| GET | `/api/system/preview` | admin | Preview a rendered page |

---

## Common Response Shapes

### Success

```json
{ "success": true, "data": { ... } }
```

### Error

```json
{ "success": false, "error": "Human-readable message", "code": "ERROR_CODE" }
```

### HTTP Status Codes Used

| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad request / validation failure |
| 401 | Missing or invalid session |
| 403 | Insufficient role or PLAC denied |
| 404 | Resource not found |
| 405 | Method not allowed on public route |
| 429 | Rate limit exceeded |
| 500 | Internal server error |

---

## Response Helpers (`src/lib/api.ts`)

| Helper | Signature |
|--------|-----------|
| `jsonOk` | `(data, status?) => Response` — sets `Content-Type: application/json` |
| `jsonError` | `(status, message, code?) => Response` — error shape |
| `withETag` | `(response, etag) => Response` — adds ETag + 304 support |

All sensitive API responses include `Cache-Control: no-store, no-cache, must-revalidate`.
