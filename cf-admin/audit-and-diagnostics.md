# cf-admin — Audit & Diagnostics

> Ghost Audit Engine, Login Forensics, System Diagnostics, and CF Access log ingestion.

---

## Ghost Audit Engine

The Ghost Audit Engine provides a tamper-evident record of every significant admin action. It is "ghost" because it runs **after** the response is sent — invisible to the user, zero latency impact.

### Design Principles

1. **Deferred execution**: Uses `ctx.waitUntil()` — audit writes happen after response delivery
2. **Immutable records**: INSERT-only. No UPDATE or DELETE on audit tables (enforced at the application layer)
3. **Non-blocking**: If the audit write fails, the original operation still succeeds
4. **Minimal PII**: IP addresses stored as SHA-256 hashes, never raw

### What Gets Audited

| Event | Table | Trigger |
|-------|-------|---------|
| Admin login (success/failure) | `admin_login_forensics` (D1) | Every authentication attempt |
| Booking status change | `admin_audit_log` (Supabase) | PATCH /api/bookings/:id/state |
| User role change | `admin_audit_log` | PATCH /api/users/:id |
| User activation/deactivation | `admin_audit_log` | PATCH /api/users/:id |
| Page access grant/revoke | `admin_audit_log` | PATCH /api/users/access |
| Feature flag toggle | `admin_audit_log` | POST /api/features/toggle |
| CMS content edit | `admin_audit_log` | PATCH /api/content/* |
| Session force-kick | `admin_audit_log` | POST /api/users/force-kick |
| ARCO document access | `admin_audit_log` | GET /api/arco/get-document |
| System diagnostic run | `admin_diagnostics_log` (D1) | POST /api/diagnostics/run |

### Audit Log Entry Structure

```typescript
// src/lib/audit.ts
interface AuditEntry {
  admin_email: string;        // Who performed the action
  action: string;             // 'BOOKING_STATUS_CHANGED', 'USER_DEACTIVATED', etc.
  resource_type: string;      // 'booking', 'user', 'feature_flag', etc.
  resource_id: string;        // The affected record's ID
  before_value?: string;      // Previous state (JSON serialized)
  after_value?: string;       // New state (JSON serialized)
  ip_hash: string;            // SHA-256(ip + IP_HASH_SECRET)
  user_agent: string;
  cf_ray: string;             // Cloudflare Ray ID for request tracing
  created_at: string;         // ISO 8601 UTC timestamp
}
```

### Audit Suppression

For DEV-only maintenance operations that would otherwise flood the audit log:

```typescript
// Toggle via /api/audit/silence (DEV role only)
session.auditSilenced = true;  // Stored in KV session
```

When `auditSilenced` is true, `auditLog()` calls are no-ops. The flag expires with the session (24h max). This is intended for database migration runs or bulk testing — not for production use.

---

## Login Forensics

Every authentication attempt is recorded in `admin_login_forensics` (D1) with rich security metadata.

### Recorded Fields (Tier 1 — Server-Trusted Data)

These come directly from Cloudflare infrastructure, not from the browser, and cannot be spoofed:

| Field | Source | Description |
|-------|--------|-------------|
| `email` | CF-Access-JWT claim | Authenticated email (or attempted email) |
| `event_type` | cf-admin code | `LOGIN_SUCCESS`, `LOGIN_FAILED`, `LOGIN_BLOCKED` |
| `failure_reason` | cf-admin code | `NOT_WHITELISTED`, `INACTIVE_ACCOUNT`, `JWT_INVALID` |
| `ip_hash` | `request.headers.get('CF-Connecting-IP')` (hashed) | Origin IP, SHA-256 hashed |
| `country` | `request.cf.country` | 2-letter country code |
| `city` | `request.cf.city` | City name |
| `asn` | `request.cf.asn` | Autonomous System Number |
| `cf_ray` | `request.headers.get('CF-Ray')` | Cloudflare request ID |
| `bot_score` | `request.cf.botManagement?.score` | 0–99 (0 = bot, 99 = human) |
| `tls_version` | `request.cf.tlsVersion` | TLS cipher suite |
| `http_protocol` | `request.cf.httpProtocol` | HTTP/1.1, HTTP/2, HTTP/3 |
| `rtt_ms` | `request.cf.clientTcpRtt` | Round-trip time in ms |
| `identity_provider` | JWT `iss` claim parsed | `google` or `github` |
| `cf_sub_id` | JWT `sub` claim | Persistent CF Access user identifier |
| `cf_jwt_tail` | Last 8 chars of JWT | For cross-referencing with CF Access logs |

### Security Alert Emails

Failed login attempts (especially from unwhitelisted emails) trigger an immediate alert email via Resend to `SECURITY_ALERT_EMAIL`:

```
Subject: [ALERT] Failed admin login attempt
Body:
  Email: unknown@example.com
  Reason: NOT_WHITELISTED
  Country: RU (Moscow)
  Bot Score: 3 (likely bot)
  Time: 2026-05-08 14:32:11 UTC
  CF Ray: abc123def456
```

### Login Forensics UI

Viewable at `/dashboard/logs` under the "Login Security" tab:
- Searchable by email, country, event type
- Sortable by time, bot score
- IP hash shown (not raw IP — for privacy compliance)
- CF Ray links for cross-referencing with Cloudflare dashboard

---

## Activity Center

The `/dashboard/logs` page is a 4-tab audit command center:

| Tab | Data Source | What It Shows |
|-----|-------------|---------------|
| **Activity Log** | Supabase `admin_audit_log` | All admin actions with before/after values |
| **Consent Trail** | Supabase `booking_consents` | Customer consent records and privacy receipts |
| **Email Log** | Supabase `email_audit_log` | Email delivery status (queued/sent/failed/delivered) |
| **Login Forensics** | D1 `admin_login_forensics` | Authentication attempts with security metadata |

**Export**: The Activity Log tab supports CSV export (rate limited to 5 exports/hour per admin, DEV/Owner only).

**Log retention**: 
- Activity/Login logs: kept indefinitely (hotel's legal obligation)
- Email logs: 90-day rolling retention (configured in Supabase)
- Older records can be pruned via `/api/audit/prune` (DEV only, requires explicit confirmation)

---

## System Diagnostics

The `/dashboard/debug/diagnostics` page runs a comprehensive health check of all connected services.

### Health Check Architecture

All 7 component checks run simultaneously via `Promise.allSettled()` — a single slow check cannot block the others:

```typescript
// src/pages/api/diagnostics/run.ts
const [d1, supabase, upstash, chatbot, r2, sentry, queue] = 
  await Promise.allSettled([
    checkD1(env),          // SELECT 1 FROM admin_pages LIMIT 1
    checkSupabase(env),    // SELECT count(*) FROM admin_authorized_users
    checkUpstash(env),     // PING
    checkChatbot(env),     // GET {CHATBOT_WORKER_URL}/api/health
    checkR2(env),          // HEAD madagascar-images/health-check.txt
    checkSentry(env),      // Verify DSN format + attempt connection
    checkQueue(env),       // Implicit (if binding exists, queue is healthy)
  ]);
```

### Result Interpretation

| Status | Meaning |
|--------|---------|
| `healthy` | Check completed within timeout with expected response |
| `degraded` | Check completed but response was unexpected |
| `unreachable` | Check timed out or returned network error |
| `unknown` | Check threw an unexpected exception |

Results are stored in D1 `admin_diagnostics_log` for historical tracking (viewable in the Diagnostics History page).

**Timeout per check**: 5 seconds (components that exceed this are marked `unreachable` — the UI does not hang).

---

## Cloudflare Access Audit Log Ingestion

cf-admin runs a **5-minute cron trigger** (`*/5 * * * *`) to pull authentication events from the Cloudflare Zero Trust API and persist them in D1.

### Cron Flow

```
1. Cron fires (every 5 minutes)
2. cf-admin calls Cloudflare API:
   GET https://api.cloudflare.com/client/v4/accounts/{CF_ACCOUNT_ID}/access/logs/access-requests
   Authorization: Bearer {CF_API_TOKEN_READ_LOGS}
3. Parse events: filter for application = secure.madagascarhotelags.com
4. For each new event (since last poll):
   a. Upsert into D1 admin_login_forensics (idempotent on cf_ray)
   b. If LOGIN_FAILED: send Resend security alert
5. Update D1 last_poll_timestamp
```

**Why this matters**: CF Access logs include authentication events that happen at the edge before any Worker code runs — including blocked attempts that never reach cf-admin. This gives complete visibility into who tried to access the admin portal.

**Required secrets**: `CF_API_TOKEN_READ_LOGS` (read-only scoped to Zero Trust audit logs), `CF_ACCOUNT_ID` (wrangler.toml var).
