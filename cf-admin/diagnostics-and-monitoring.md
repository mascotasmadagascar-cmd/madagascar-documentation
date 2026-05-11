# cf-admin — Diagnostics & Monitoring

Diagnostic test suite (connectivity, functional, security), run lifecycle, API endpoints, Sentry integration, and monitoring gaps.

---

## Diagnostics Portal (`/dashboard/debug/diagnostics`)

The diagnostics portal provides real-time and historical visibility into the health of the entire Madagascar infrastructure. Access requires the `dev` role.

---

## Diagnostic Test Suite (`src/lib/diagnostics/`)

The suite is composed of three test categories. All tests within a category run concurrently via `Promise.allSettled()` — a single slow or failing test does not block the others.

### Connectivity Tests

Verify that all external services and bindings are reachable from the Worker:

| Test | Method | What Is Measured |
|------|--------|-----------------|
| D1 database ping | `SELECT 1` via D1 binding | Reachability and query latency |
| R2 bucket connectivity | List objects via R2 binding | R2 binding operational |
| KV namespace read/write | PUT + GET + DELETE cycle | Full KV round-trip functional |
| Supabase REST API ping | REST health endpoint via `fetch` | Supabase reachable from Worker |
| Upstash Redis connection | Redis PING command | Rate limit store reachable |

### Functional Tests

Verify that core application logic works end-to-end:

| Test | What Is Verified |
|------|-----------------|
| Session creation + retrieval roundtrip | Write AdminSession to KV → read back → compare fields |
| PLAC access map computation | D1 JOIN query executes; access map is valid |
| JWT verification (dummy token) | `verifyZeroTrustJwt()` correctly rejects a malformed token |
| Rate limiter functionality | Upstash sliding window correctly increments counter |

### Security Tests

Verify that security controls are active and functional:

| Test | What Is Verified |
|------|-----------------|
| CSRF token validation | `csrf.ts` rejects request with invalid/missing Origin header |
| Rate limit enforcement | Endpoint returns 429 after configured threshold is exceeded |
| Page access restriction | PLAC correctly denies access to a path the test user has no rights to |
| Role hierarchy enforcement | `hasPermission()` denies a lower-privileged role the required higher-privileged action |

---

## Diagnostic Run Lifecycle

```
1. Admin clicks "Run Diagnostics" in SystemDiagnostics.tsx
2. Browser sends: POST /api/diagnostics/run
3. Worker generates a run_id (UUID)
4. Worker returns immediately: { run_id, status: "started" }
      ↓ (parallel, via ctx.waitUntil)
5. runner.ts executes all three test categories concurrently
6. Each test result shape:
   {
     category: "connectivity" | "functional" | "security",
     name: "D1 database ping",
     status: "pass" | "fail" | "degraded",
     latencyMs: 2,
     error?: "Error message if failed"
   }
7. Overall status computed:
   - "pass"     → all tests passed
   - "degraded" → all completed but some showed warnings or elevated latency
   - "fail"     → one or more tests failed or threw
8. Results written to D1 admin_diagnostic_runs:
   { id, run_id, test_results (JSON), overall_status, created_at }
9. UI polls GET /api/diagnostics/results or uses run_id to fetch
```

---

## D1 Results Schema

```sql
CREATE TABLE admin_diagnostic_runs (
  id TEXT PRIMARY KEY,
  run_id TEXT,
  test_results TEXT,        -- JSON array of individual test result objects
  overall_status TEXT,      -- pass | fail | degraded
  created_at TEXT
);
```

---

## Diagnostic API Endpoints

| Method | Path | Min Role | Description |
|--------|------|----------|-------------|
| GET | `/api/diagnostics/ping` | — | Simple liveness check — returns `{ "ok": true }` |
| GET | `/api/health` | — | Public health check with D1, KV, R2, Supabase latency |
| POST | `/api/diagnostics/run` | dev | Start full diagnostic suite; returns run_id immediately |
| GET | `/api/diagnostics/results` | dev | Fetch all historical diagnostic runs from D1 |
| GET | `/api/diagnostics/infrastructure` | admin | Real-time infrastructure status (not persisted) |

### `GET /api/health` Response Shape

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

### `GET /api/diagnostics/results` Response Shape

```json
{
  "data": [
    {
      "id": "hex16",
      "run_id": "uuid",
      "overall_status": "pass",
      "created_at": "2026-05-10T12:00:00Z",
      "test_results": [
        {
          "category": "connectivity",
          "name": "D1 database ping",
          "status": "pass",
          "latencyMs": 2
        },
        {
          "category": "security",
          "name": "CSRF token validation",
          "status": "pass",
          "latencyMs": 0
        }
      ]
    }
  ]
}
```

---

## Diagnostics UI Components

| Component | Purpose |
|-----------|---------|
| `SystemDiagnostics.tsx` | Main panel — trigger runs, show current status |
| `DiagnosticsTestList.tsx` | Table of all test results with pass/fail indicators per category |
| `DiagnosticsInfraBar.tsx` | Compact health strip shown at the top of debug pages |
| `SystemDiagnosticsHistory.tsx` | Historical run browser — view all past runs from D1 |
| `PageRegistryManager.tsx` | View and toggle `is_active` on pages in `admin_pages` |

---

## Performance Benchmarks (`src/lib/diagnostics/benchmarks.ts`)

The benchmarks module collects latency measurements during diagnostic runs and includes them in the run report:

- D1 query latency (single-statement and JOIN)
- KV read and write round-trip latency
- Supabase query latency
- Upstash Redis command latency
- CHATBOT_SERVICE internal call latency
- ASTRO_SERVICE internal call latency

Benchmark results are appended to the `test_results` JSON array in `admin_diagnostic_runs`.

---

## Sentry Integration

### Configuration

```typescript
// sentry.server.config.ts
import * as Sentry from '@sentry/astro';

Sentry.init({
  dsn: env.PUBLIC_SENTRY_DSN,  // https://389bb...@o4510752...ingest.us.sentry.io/4511193...
  tracesSampleRate: 1.0,        // 100% sampling — all admin operations traced
});
```

### What Sentry Captures

- Unhandled exceptions in middleware and API routes
- Outgoing subrequest errors (Supabase, Resend, Upstash)
- Source maps uploaded during CI/CD build (requires `SENTRY_AUTH_TOKEN`)
- Distributed traces across the request lifecycle

### Session Context Enrichment

The middleware sets Sentry user context after session retrieval:

```typescript
if (session) {
  Sentry.setUser({
    id: session.userId,
    email: session.email,
    // role intentionally omitted from Sentry context (PII minimization)
  });
}
```

### CI/CD Source Map Upload

Source maps are uploaded during `npm run cf:deploy` using:
- `SENTRY_AUTH_TOKEN` (CI secret)
- `SENTRY_ORG_SLUG`
- `SENTRY_PROJECT_SLUG`

These are not required at runtime — only at build time.

---

## Monitoring Gaps

The following monitoring capabilities are **not** currently implemented in cf-admin:

| Gap | Notes |
|-----|-------|
| BetterStack / Logtail | cf-astro uses Logtail; cf-admin uses Sentry only |
| Uptime monitoring | No external uptime check configured for `secure.madagascarhotelags.com` |
| Alerting on high error rates | Sentry captures errors but no alert thresholds are configured |
| Analytics Engine dashboards | The `ANALYTICS` binding exists but no custom dashboard is built |
| Cron failure alerting | If the CF Access audit cron fails silently, there is no alert |

---

## Infrastructure Status Components

The `GET /api/diagnostics/infrastructure` endpoint (min role: `admin`) returns real-time status without persisting to D1. Used by `DiagnosticsInfraBar.tsx` to show the health strip on debug pages.

Unlike `POST /api/diagnostics/run`, this endpoint is lightweight — it only checks binding reachability (D1 ping, KV get, R2 list), not functional or security tests.
