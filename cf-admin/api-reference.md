# cf-admin — API Reference

> Internal REST API endpoints for the dashboard modules.

---

## Authentication

All API endpoints (except `/api/auth/dev-login`) require a valid session in `SESSION_KV`. The middleware pipeline checks this and rejects unauthorized requests with a `403 Forbidden` status.

---

## 1. Authentication Endpoints

### `POST /api/auth/dev-login`
Creates a local development session (bypasses CF Access). Only available when `SITE_URL` contains `localhost`.

**Request Body:**
```json
{
  "email": "dev@example.com",
  "name": "Dev User"
}
```

**Response (200):**
```json
{
  "success": true,
  "redirect": "/dashboard"
}
```
*Side Effect:* Sets `SESSION_ID` cookie.

### `POST /api/auth/logout`
Destroys the current session and redirects to login.

**Response (200):**
```json
{
  "success": true,
  "redirect": "/"
}
```
*Side Effect:* Deletes `SESSION_ID` cookie and removes KV session.

---

## 2. Bookings Management

### `GET /api/bookings`
Fetch list of bookings. Can be filtered/paginated.

**Query Parameters:**
- `status` (optional): Filter by status (e.g., `pending`, `confirmed`)
- `limit` (optional): Default 50
- `offset` (optional): Default 0

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
      "status": "pending",
      "booking_ref": "MDG-2026-0042",
      "created_at": "..."
    }
  ],
  "total": 120
}
```

### `GET /api/bookings/:id`
Fetch a specific booking with pets and consent details.

**Response (200):**
```json
{
  "data": {
    "booking": { /* booking details */ },
    "pets": [ /* array of pets */ ],
    "consents": [ /* array of consents */ ]
  }
}
```

### `PATCH /api/bookings/:id`
Update a booking (e.g., change status, update notes).

**Request Body:**
```json
{
  "status": "confirmed",
  "notes": "VIP guest"
}
```

**Response (200):**
```json
{
  "success": true,
  "data": { /* updated booking */ }
}
```
*Side Effect:* Ghost audit log entry created.

---

## 3. User Management (RBAC)

### `GET /api/users`
Fetch list of authorized admin users.

**Response (200):**
```json
{
  "data": [
    {
      "id": "uuid",
      "email": "admin@example.com",
      "role": "admin",
      "display_name": "Admin User",
      "is_active": true
    }
  ]
}
```

### `PATCH /api/users/:id`
Update a user's role or status. Requires higher privilege than the target user.

**Request Body:**
```json
{
  "role": "editor",
  "is_active": true
}
```

**Response (200):**
```json
{
  "success": true
}
```
*Side Effect:* Target user's session is revoked, forcing re-authentication.

---

## 4. CMS & Feature Flags

### `GET /api/cms/flags`
Fetch all feature flags from D1.

**Response (200):**
```json
{
  "data": [
    {
      "flag_key": "new_booking_ui",
      "is_enabled": 1,
      "description": "Enable V2 booking flow"
    }
  ]
}
```

### `PATCH /api/cms/flags/:key`
Toggle a feature flag.

**Request Body:**
```json
{
  "is_enabled": 0
}
```

**Response (200):**
```json
{
  "success": true
}
```
*Side Effect:* D1 updated. Public site will pick up changes on next cache miss (max 60s delay).

### `POST /api/cms/revalidate`
Trigger ISR revalidation on the public site.

**Request Body:**
```json
{
  "paths": ["/services", "/en/services"]
}
```

**Response (200):**
```json
{
  "success": true,
  "invalidated": 2
}
```
*Side Effect:* Proxies request to `cf-astro/api/revalidate`.

---

## 5. Chatbot Proxy

### `GET /api/chatbot/kb`
Fetch Knowledge Base entries. Proxied to `cf-chatbot`.

**Response (200):**
```json
{
  "data": [
    {
      "id": 1,
      "title": "Check-in policies",
      "category": "policies",
      "content": "..."
    }
  ]
}
```

### `POST /api/chatbot/kb`
Add or update a KB entry. Proxied to `cf-chatbot`.

**Request Body:**
```json
{
  "title": "New Policy",
  "category": "policies",
  "content": "Policy details..."
}
```

**Response (200):**
```json
{
  "success": true,
  "id": 2
}
```
*Side Effect:* `cf-chatbot` will automatically generate embeddings and insert into Vectorize.

---

## 6. Diagnostics

### `GET /api/diagnostics/health`
Run comprehensive system health checks.

**Response (200):**
```json
{
  "status": "healthy",
  "components": {
    "supabase": { "status": "up", "latency": 35 },
    "d1": { "status": "up", "latency": 2 },
    "kv": { "status": "up", "latency": 4 },
    "cf_astro": { "status": "up", "latency": 45 },
    "cf_chatbot": { "status": "up", "latency": 50 }
  }
}
```

### `GET /api/diagnostics/logs`
Fetch recent audit logs from D1.

**Response (200):**
```json
{
  "data": [
    {
      "id": 1,
      "user_email": "admin@example.com",
      "action": "view",
      "module": "bookings",
      "created_at": "..."
    }
  ]
}
```
