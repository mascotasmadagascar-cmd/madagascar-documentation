# cf-admin — Audit & Diagnostics

Ghost Audit Engine, login forensics, CF Access log ingestion via cron, and the diagnostics test suite.

---

## Ghost Audit Engine (`src/lib/audit.ts`, 134 lines)

### Design

The Ghost Audit Engine logs every significant admin action with **zero latency impact** on the request path. The name "ghost" reflects that writes happen after the response is already sent.

**Core pattern:**
```typescript
// ctx.waitUntil() — Cloudflare continues executing this promise after
// the HTTP response is returned to the browser
ctx.waitUntil(
  auditLog(waitUntil, db, {
    userId, userEmail, userRole,
    action, module, targetId, targetType,
    details,   // JSON object — free-form context
    ipHash     // HMAC(IP, IP_HASH_SECRET) — privacy-safe
  }, silent)
)
```

**Design principles:**
- INSERT-only — no UPDATE or DELETE on audit tables (enforced at application layer)
- Non-blocking — audit write failure does not affect the user's operation
- Privacy-safe — IP stored as HMAC hash, never raw
- Fire-and-forget — zero perceived latency

### D1 Table Schema

```sql
CREATE TABLE admin_audit_log (
  id TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
  user_id TEXT,
  user_email TEXT,
  user_role TEXT,
  action TEXT,       -- See action enum below
  module TEXT,       -- See module enum below
  target_id TEXT,    -- Who or what was affected (UUID, path, flag key, etc.)
  target_type TEXT,  -- Type descriptor (user, page, booking, cms_block, etc.)
  details TEXT,      -- JSON object with free-form context
  ip_hash TEXT,      -- HMAC(IP, IP_HASH_SECRET) — never raw IP
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_audit_user   ON admin_audit_log(user_id,   created_at DESC);
CREATE INDEX idx_audit_module ON admin_audit_log(module,    created_at DESC);
```

### Action Enum

| Action | When Used |
|--------|-----------|
| `login` | Successful authentication |
| `logout` | Session destroyed (voluntary) |
| `view` | Dashboard page viewed |
| `create` | Record created (user, booking state, CMS block) |
| `update` | Record updated |
| `delete` | Record deleted |
| `grant_access` | PLAC explicit grant added |
| `revoke_access` | PLAC explicit deny added |
| `reset_access` | PLAC override removed (revert to role default) |
| `force_logout` | Force-kick executed on a user |
| `export` | Data export initiated |
| `prune` | Old audit records deleted |
| `registry_update` | Page registry `is_active` toggled |

### Module Enum

| Module | Covers |
|--------|--------|
| `auth` | Login, logout, session events |
| `plac` | Page override grant/revoke/reset |
| `users` | User create, update, delete, force-kick |
| `content` | CMS block edits, image uploads |
| `bookings` | Booking state changes |
| `media` | R2 image uploads |
| `debug` | Diagnostic runs, feature flag changes |
| `system` | Page registry changes |
| `settings` | Portal and user settings changes |
| `audit` | Audit prune, audit silence toggle |
| `chatbot` | Chatbot config, KB edits via proxy |

### API

```typescript
// Direct call
await auditLog(waitUntil, db, {
  userId, userEmail, userRole,
  action, module, targetId, targetType,
  details, ipHash
}, silent)

// Bound logger (preferred in API routes)
const log = createAuditLogger({
  db,
  waitUntil,
  tableName: 'admin_audit_log',
  silenced: session.auditSilenced ?? false
})
await log({ action: 'update', module: 'users', targetId: userId, ... })
```

### Audit Silence (DEV-only)

When `session.auditSilenced = true`, all `auditLog()` calls are no-ops. Used during testing or bulk migrations to avoid polluting the audit trail.

Toggled via `POST /api/audit/silence` (DEV role required). Stored in the KV session. Expires automatically with the session (24-hour max lifetime).

```json
// Request
{ "silenced": true }

// Effect: session.auditSilenced = true — all subsequent audit writes are skipped
```

---

## Login Forensics (`src/lib/auth/security-logging.ts`, 344 lines)

Every authentication attempt is recorded in D1 `admin_login_logs` with rich security metadata sourced from Cloudflare infrastructure (Tier 1 — server-trusted, cannot be spoofed by the browser).

### D1 Table Schema

```sql
CREATE TABLE admin_login_logs (
  id TEXT PRIMARY KEY,
  email TEXT,
  event_type TEXT,              -- LOGIN_SUCCESS | LOGIN_FAILED | LOGIN_BLOCKED
  success INTEGER,              -- 0 or 1
  is_authorized_email INTEGER,  -- 1=in whitelist, 0=not in whitelist
  failure_reason TEXT,          -- not_whitelisted | account_inactive | cf_access_denied
  ip_address TEXT,              -- Raw IP (internal use only, not shown in UI)
  user_agent TEXT,              -- Truncated to 512 chars
  geo_location TEXT,            -- "city, country" display string
  login_method TEXT,            -- google | github | otp
  created_at TEXT,

  -- Tier 1: CF request.cf geo data (server-trusted, not from browser headers)
  latitude TEXT,
  longitude TEXT,
  postal_code TEXT,
  timezone TEXT,
  continent TEXT,
  asn INTEGER,
  asn_org TEXT,
  colo TEXT,                    -- Cloudflare colocation code (e.g., "LAX", "MAD")
  tls_version TEXT,
  http_protocol TEXT,           -- h2, h3
  client_rtt_ms INTEGER,        -- TCP round-trip time in milliseconds

  -- CF Zero Trust fields (from verified JWT)
  cf_ray_id TEXT,               -- CF-Ray header — links to Cloudflare dashboard trace
  cf_access_method TEXT,        -- google | github | otp
  cf_identity_provider TEXT,    -- google-oauth2, github, otp (idp.type claim)
  cf_jwt_tail TEXT,             -- Last 16 chars of JWT (audit reference, not secret)
  cf_bot_score INTEGER          -- CF Bot Management score (if available)
);
```

### Event Types

| Event Type | Trigger | Alert Email Sent? |
|------------|---------|-------------------|
| `LOGIN_SUCCESS` | CF Access JWT verified + user in whitelist + `is_active = 1` | No |
| `LOGIN_FAILED` | CF Access JWT invalid or expired (before whitelist check) | No |
| `LOGIN_BLOCKED` | JWT valid but user not in whitelist OR `is_active = 0` | Yes |

### Security Alert Emails

On `LOGIN_BLOCKED` events, an HTML alert email is sent via Resend:
- Dispatched async via `ctx.waitUntil()` — does not block the 403 response
- Recipient: `ADMIN_EMAIL` (`mascotasmadagascar@gmail.com`)
- Sender: `team@madagascarhotelags.com` via `RESEND_API_KEY`
- Email template stored in `email-templates/` directory

**Email content includes:**
- Attempted email address
- Login method (google / github / otp)
- Timestamp (UTC)
- Masked IP address
- Geo location (city, country)
- CF Ray ID (for cross-referencing with Cloudflare dashboard)
- User agent string
- CF Bot Management score (if available)

---

## CF Access Audit Log Cron (`src/workers/scheduled-log-sync.ts`)

CF Access logs authentication events at the Cloudflare edge — including blocked attempts that never reach the Worker. The cron imports these into D1 for the login forensics UI.

### Cron Schedule

```toml
[triggers]
crons = ["*/5 * * * *"]   # Every 5 minutes — 288 invocations per day
```

### Cron Flow

```
1. handleScheduled() fires every 5 minutes
2. Read last-synced timestamp from KV: cf-audit-last-synced
3. GET https://api.cloudflare.com/client/v4/accounts/{CF_ACCOUNT_ID}/access/logs/access_requests
   Authorization: Bearer {CF_API_TOKEN_READ_LOGS}
   (reads only events since last-synced timestamp)
4. Filter events for application AUD: CF_ACCESS_AUD
5. For each new event:
   a. Upsert into D1 admin_login_logs (idempotent on cf_ray_id)
   b. If event type is LOGIN_BLOCKED: dispatch security alert email via Resend
6. Write current ISO timestamp to KV: cf-audit-last-synced
```

**Required bindings/secrets:**
- `CF_API_TOKEN_READ_LOGS` — read-only scope: `Access: Audit Logs: Read`
- `CF_ACCOUNT_ID` — `320d1ebab5143958d2acd481ea465f52` (in `wrangler.toml [vars]`)
- `SESSION` KV namespace (for `cf-audit-last-synced` key)
- `DB` D1 database

---

## Audit Log Activity Center (UI)

The `/dashboard/logs` page is a multi-tab audit command center:

| Tab | Component | Data Source | Contents |
|-----|-----------|-------------|----------|
| Activity Log | `ActivityLogTab.tsx` | D1 `admin_audit_log` | All admin actions — paginated, filterable |
| Login Forensics | `LoginForensicsTab.tsx` | D1 `admin_login_logs` | Full forensics: IP hash, geo, user-agent, CF fields |
| Email Log | `EmailLogTab.tsx` | D1 or Supabase | Email delivery status and audit trail |
| Consent Trail | `ConsentTrailTab.tsx` | D1 or Supabase | Customer consent records and privacy receipts |

**Log retention:**
- Audit and login logs: kept indefinitely (legal obligation)
- Old records can be pruned via `POST /api/audit/prune` (DEV role required, per retention policy)

---

## Diagnostics System (`src/lib/diagnostics/`)

The diagnostics portal (`/dashboard/debug/diagnostics`) runs a comprehensive health check suite across all infrastructure components.

### Architecture

- All tests run via `runner.ts` which orchestrates test categories using `Promise.allSettled()`
- A single slow or failing test does not block others
- Results are persisted to D1 `admin_diagnostic_runs` via `ctx.waitUntil()`
- Historical results viewable via `GET /api/diagnostics/results`

### Test Categories

#### Connectivity Tests

| Test | Method | What Is Verified |
|------|--------|------------------|
| D1 database ping | `SELECT 1` via D1 binding | Database reachable, measures latency |
| R2 bucket connectivity | List objects | R2 binding functional |
| KV namespace read/write | PUT + GET + DELETE | KV round-trip functional |
| Supabase REST API ping | REST health endpoint | Supabase reachable from Worker |
| Upstash Redis connection | Redis PING command | Rate limit store reachable |

#### Functional Tests

| Test | What Is Verified |
|------|-----------------|
| Session creation + retrieval roundtrip | KV write → read → compare |
| PLAC access map computation | D1 JOIN query runs correctly |
| JWT verification (dummy token) | `verifyZeroTrustJwt` logic |
| Rate limiter functionality | Upstash sliding window increments |

#### Security Tests

| Test | What Is Verified |
|------|-----------------|
| CSRF token validation | `csrf.ts` rejects invalid tokens |
| Rate limit enforcement | Endpoint correctly returns 429 after threshold |
| Page access restriction | PLAC denies access to restricted path |
| Role hierarchy enforcement | `hasPermission()` denies lower roles correctly |

### D1 Results Schema

```sql
CREATE TABLE admin_diagnostic_runs (
  id TEXT PRIMARY KEY,
  run_id TEXT,
  test_results TEXT,       -- JSON array of individual test outcomes
  overall_status TEXT,     -- pass | fail | degraded
  created_at TEXT
);
```

### Diagnostic API Endpoints

| Method | Path | Min Role | Description |
|--------|------|----------|-------------|
| POST | `/api/diagnostics/run` | dev | Execute full suite — returns run_id immediately |
| GET | `/api/diagnostics/results` | dev | Fetch all historical runs from D1 |
| GET | `/api/diagnostics/infrastructure` | admin | Real-time infrastructure status (no D1 persistence) |
| GET | `/api/diagnostics/ping` | — | Simple liveness — `{ "ok": true }` |
| GET | `/api/health` | — | Public health check with component latency |

### Result Status Values

| Status | Meaning |
|--------|---------|
| `pass` | All checks completed with expected responses |
| `fail` | One or more checks returned unexpected response or error |
| `degraded` | All checks completed but some showed elevated latency or warnings |

### UI Components

| Component | Purpose |
|-----------|---------|
| `SystemDiagnostics.tsx` | Main diagnostics runner panel |
| `DiagnosticsTestList.tsx` | Test result table with pass/fail indicators |
| `DiagnosticsInfraBar.tsx` | Top-of-page health indicator strip |
| `SystemDiagnosticsHistory.tsx` | Historical run browser |

---

## Performance Benchmarks (`src/lib/diagnostics/benchmarks.ts`)

The benchmarks module collects performance metrics during diagnostic runs:

- D1 query latency (p50, p95)
- KV read/write latency
- Supabase query latency
- Upstash Redis latency
- CF service binding round-trip latency (CHATBOT_SERVICE, ASTRO_SERVICE)

Results are included in the `test_results` JSON of each diagnostic run.
