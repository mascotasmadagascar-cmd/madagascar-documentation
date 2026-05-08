# cf-admin — Diagnostics & Monitoring

> System health checks, infrastructure dashboards, and Sentry integration.

---

## The Diagnostics Portal (`/dashboard/debug/diagnostics`)

cf-admin includes a built-in infrastructure diagnostics portal designed for developers and system administrators (requires `dev` role). It provides real-time visibility into the health of the entire Madagascar ecosystem.

### Key Features
1. **Real-time Status Matrix**: Red/Green status indicators for all external dependencies (Supabase, Upstash, Resend).
2. **Internal Service Health**: Connectivity checks between cf-admin, cf-astro, and cf-chatbot.
3. **Cloudflare Resource Monitor**: Live read/write operational status for D1, KV, and Vectorize bindings.
4. **Audit Log Viewer**: UI for browsing D1-backed security audit logs and Cloudflare Access logs.

---

## Health Check Implementation

The `/api/diagnostics/health` endpoint aggregates status from multiple sources using `Promise.allSettled` to prevent a single slow service from blocking the report.

### Check Types

| Component | Check Method | Timeout |
|-----------|--------------|---------|
| **Supabase** | `SELECT 1` via Supabase JS | 3s |
| **D1** | `SELECT 1` via binding | 1s |
| **KV** | `get('health_check')` via binding | 1s |
| **Upstash** | `GET` health endpoint via `fetch` | 3s |
| **Resend** | `GET /emails` (list limit=1) via `fetch` | 3s |
| **cf-astro** | `GET {SITE_URL}/api/health` | 2s |
| **cf-chatbot** | `GET {CHATBOT_URL}/api/health` | 2s |

### Example Response Payload
```json
{
  "status": "degraded",
  "timestamp": "2026-05-08T05:00:00Z",
  "components": {
    "supabase": { "status": "up", "latencyMs": 45 },
    "d1": { "status": "up", "latencyMs": 2 },
    "kv": { "status": "up", "latencyMs": 5 },
    "cf_astro": { "status": "up", "latencyMs": 30 },
    "cf_chatbot": { "status": "down", "error": "Connection timeout" },
    "upstash": { "status": "up", "latencyMs": 15 },
    "resend": { "status": "up", "latencyMs": 120 }
  }
}
```

---

## Log Ingestion (Cloudflare Access)

A crucial security feature is the ingestion of Cloudflare Access authentication logs into the administrative dashboard.

### The Cron Flow

1. **Trigger**: `crons = ["*/30 * * * *"]` (Wrangler configuration).
2. **Fetch**: The worker calls the Cloudflare API (`https://api.cloudflare.com/client/v4/accounts/{account_id}/access/logs/access_requests`).
3. **Filter**: Extracts logs specific to the cf-admin application (`CF_ACCESS_AUD`).
4. **Store**: Inserts new records into the D1 `cf_access_audit_log` table.

This allows administrators to see failed login attempts and unauthorized access attempts directly within the cf-admin UI, without needing to log into the Cloudflare dashboard.

---

## Sentry Integration

cf-admin uses Sentry for error tracking and distributed tracing.

### Configuration
```typescript
// sentry.server.config.ts
import * as Sentry from '@sentry/cloudflare';

Sentry.init({
  dsn: env.PUBLIC_SENTRY_DSN,
  tracesSampleRate: 1.0, // 100% sampling for admin operations
});
```

### Automatic Instrumentation
The `@sentry/cloudflare` SDK automatically wraps the `fetch` handler in `src/index.ts`, capturing:
- Unhandled exceptions
- Outgoing HTTP requests (Supabase, Resend)
- Request metadata (headers, URL, method)

### Custom Context
The authentication middleware enriches the Sentry context with user data:
```typescript
// middleware.ts
if (session) {
  Sentry.setUser({
    id: session.userId,
    email: session.email,
    role: session.role
  });
}
```

---

## Analytics Engine

While cf-astro uses Analytics Engine for public traffic telemetry, cf-admin uses it for internal usage metrics.

### Tracking Admin Activity
```typescript
env.ANALYTICS.writeDataPoint({
  blobs: [
    'admin_action',
    session.role,
    url.pathname,
    request.method
  ],
  doubles: [1],
  indexes: [session.userId], // Indexed by user ID for fast querying
});
```

These metrics help identify which dashboard modules are actively used and which roles perform the most actions.
