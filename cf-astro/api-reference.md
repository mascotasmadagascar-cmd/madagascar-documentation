# cf-astro — API Reference

> All REST API endpoints exposed by the public website.

---

## Authentication
Public API endpoints use **Cloudflare Turnstile** CAPTCHA for bot protection and **Upstash rate limiting** for abuse prevention. No user authentication is required (public site).

---

## Endpoints

### `POST /api/booking`

Submit a new pet hotel booking.

**Rate Limit**: 5 requests per hour per IP

**Request Body**:
```json
{
  "guest_name": "María García",
  "email": "maria@example.com",
  "phone": "+524491234567",
  "check_in": "2026-06-01",
  "check_out": "2026-06-05",
  "special_requests": "Late check-in, please",
  "turnstile_token": "0.xxx...",
  "pets": [
    {
      "name": "Luna",
      "species": "dog",
      "breed": "Golden Retriever",
      "weight": 30,
      "age": 3,
      "special_needs": "Needs daily medication",
      "vaccination_status": "up_to_date"
    }
  ],
  "consents": {
    "terms": true,
    "privacy": true,
    "marketing_email": false
  }
}
```

**Validation**: Zod schema validates all fields. Turnstile token verified server-side.

**Response (200)**:
```json
{
  "success": true,
  "bookingRef": "MDG-2026-0042",
  "trackingId": "e5f4a3b2-1234-5678-9abc-def012345678",
  "message": "Booking submitted successfully"
}
```

**Response (429)**:
```json
{
  "error": "Too many requests. Please try again later."
}
```

**Side Effects**:
1. Inserts `bookings` + `pets` + `booking_consents` rows in Supabase
2. Inserts `email_audit_log` row with `tracking_id`
3. Produces 2 Queue messages (admin notification + customer confirmation)
4. Writes Analytics Engine data point

---

### `POST /api/contact`

Submit a contact form inquiry.

**Rate Limit**: 3 requests per hour per IP

**Request Body**:
```json
{
  "name": "Carlos López",
  "email": "carlos@example.com",
  "phone": "+524491234567",
  "message": "Do you accept large breed dogs?",
  "turnstile_token": "0.xxx..."
}
```

**Response (200)**:
```json
{
  "success": true,
  "message": "Message sent successfully"
}
```

**Side Effects**:
1. Produces Queue message (contact form email to admin)
2. Writes Analytics Engine data point

---

### `POST /api/arco`

Submit an ARCO (data privacy rights) request.

**Rate Limit**: 2 requests per day per IP

**Request Body**:
```json
{
  "name": "Ana Rodríguez",
  "email": "ana@example.com",
  "request_type": "access",
  "description": "I want to know what data you have about me",
  "turnstile_token": "0.xxx...",
  "identity_document": "(file upload via R2)"
}
```

**Response (200)**:
```json
{
  "success": true,
  "reference": "ARCO-2026-0003",
  "message": "Your request has been submitted. We will respond within 20 business days."
}
```

**Side Effects**:
1. Stores identity document in R2 (`arco-documents` private bucket)
2. Inserts ARCO request record in Supabase
3. Produces Queue message (ARCO notification to admin)

---

### `POST /api/revalidate`

Invalidate ISR cache for specific paths. Called by cf-admin on CMS updates.

**Authentication**: `Authorization: Bearer {REVALIDATION_SECRET}`

**Request Body**:
```json
{
  "paths": ["/", "/en", "/services", "/en/services"]
}
```

**Response (200)**:
```json
{
  "success": true,
  "invalidated": 4,
  "message": "Cache invalidated for 4 paths"
}
```

**Side Effects**:
1. Deletes matching KV keys from `ISR_CACHE` namespace

---

### `GET /api/health`

Health check endpoint for monitoring.

**Response (200)**:
```json
{
  "status": "ok",
  "timestamp": "2026-05-08T05:00:00Z",
  "version": "1.0.0"
}
```

---

### `GET /sitemap.xml`

Auto-generated sitemap for search engines. Lists all public pages in both locales.

### `GET /robots.txt`

Search engine crawl directives. Allows all crawlers, references sitemap.
