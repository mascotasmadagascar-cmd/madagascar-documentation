# cf-admin — Cost & Scaling

> Admin portal usage patterns, free tier consumption, and scaling projections.

---

## Admin vs Public Site: Different Usage Pattern

cf-admin has fundamentally different usage characteristics than cf-astro:

| Dimension | cf-astro (public) | cf-admin |
|-----------|------------------|---------|
| **Users** | Hundreds of visitors/day | 2–5 concurrent admins max |
| **Requests/day** | ~2,000–5,000 | ~200–500 |
| **Write operations** | Low (bookings only) | High (CMS, user mgmt, audit logs) |
| **CPU per request** | ~2–5ms (ISR cache hits) | ~5–9ms (session + RBAC + PLAC) |
| **Cron traffic** | None | 288 requests/day (5-min CF audit poll) |

---

## Service-by-Service Breakdown

### Cloudflare Workers

**Requests per day** (estimated):
- Admin UI pages: ~50–100 requests (5 admins × 10–20 page loads)
- API routes (bookings, CMS, users): ~100–200 requests
- Cron trigger (*/5 * * * *): 288 requests/day
- Diagnostic health checks: ~20 requests
- **Total**: ~460–610 requests/day

**Free tier limit**: 100,000 requests/day
**Headroom**: ~99.5% — admin will never exhaust Workers free tier

**CPU per request**: 
- Session middleware: ~1–2ms (KV read)
- RBAC check: <0.1ms (in-memory)
- PLAC check: ~0.1ms (in-memory, refreshed every 60min)
- Business logic: ~2–6ms
- **Total per request**: ~3–8ms (well under 10ms limit)

---

### Cloudflare KV

The most operationally important resource to monitor.

**Reads per day**:
- Session reads: ~460–610 (one per request)
- PLAC map reads: already included in session read
- Revocation flag checks: ~460–610
- **Total reads**: ~920–1,220/day
- **Limit**: 100,000/day → **~99% headroom**

**Writes per day**:
- New admin sessions (logins): ~5–15 (per admin login)
- Role freshness updates: ~20–60 (re-written every 30 min)
- ISR cache invalidations: ~10–50 (per content edit)
- Revocation flag sets: ~0–2 (rare)
- **Total writes**: ~35–127/day
- **Limit**: **1,000 writes/day**
- **Current headroom**: ~87–96%

> ⚠️ **KV writes are the binding constraint.** At 10x admin activity (10+ concurrent admins editing content constantly), writes could approach 1,000/day. Monitor via `wrangler kv:key list --namespace-id <SESSION_ID> --remote | wc -l` periodically.

---

### D1 Database

Admin writes more heavily to D1 than cf-astro (audit logs, session forensics, CMS content).

**Writes per day**:
- Login forensics records: ~5–15 (per login event)
- Audit log entries: ~50–200 (per admin action via Ghost Audit Engine)
- CF Access audit log ingestion: ~50–100 events/day (via cron)
- CMS content updates: ~10–50
- Diagnostic log entries: ~10–20
- **Total writes**: ~125–385/day
- **Limit**: 100,000/day → **>99% headroom**

**Reads per day**:
- PLAC computation: ~5–10 (freshness: 60min TTL)
- Feature flag reads (D1 fallback only): ~5–20
- Booking list/detail: ~50–100
- User registry: ~10–20
- Audit log viewer: ~20–50
- **Total reads**: ~90–200/day
- **Limit**: 5,000,000/day → **>99.99% headroom**

---

### Cron Trigger Analysis

The `*/5 * * * *` cron runs 288 times per day. Each invocation:
1. Calls Cloudflare API (1 subrequest)
2. Processes ~0–20 new auth events (typically 0 at low activity periods)
3. Writes 0–20 rows to D1 `admin_login_forensics`
4. Sends 0–1 Resend security alert emails (only on blocked login attempts)

**Workers cost impact**: 288 requests/day added to the Workers request count. At normal scale, total Workers requests (admin + cron) stay well under 1,000/day.

---

## Scaling Projections

| Scenario | Workers req/day | KV writes/day | D1 writes/day | Cost |
|----------|----------------|---------------|---------------|------|
| Current (2–5 admins) | ~600 | ~100 | ~300 | **$0** |
| 10 admins | ~1,500 | ~350 | ~800 | **$0** |
| 20 admins + heavy editing | ~5,000 | ~800 | ~2,000 | **$0** |
| 50 admins (hotel chain scale) | ~15,000 | ~2,500 | ~5,000 | Workers Std $5/mo; KV upgrade needed |

**First bottleneck at admin scale**: KV writes (~1,000/day limit) would be reached at ~20 active admins editing content simultaneously. Upgrade to Workers Standard ($5/mo) raises KV write limit to 1,000,000/month.

---

## Optimization Notes

**Session KV efficiency**: The session is read once per request and cached in the Worker's execution context. Middleware passes the session object forward — API routes do not re-read KV.

**PLAC computation cost**: PLAC maps are computed once per 60 minutes per admin (D1 query), then served from the KV session. This amortizes the D1 read cost across all requests within the 60-minute window.

**Ghost Audit Engine**: Uses `ctx.waitUntil()` — audit writes never add to the user-perceived request latency. Even if D1 is slow, the admin's response arrives immediately.

**Cron efficiency**: The CF Access audit poll cron uses cursor-based pagination to avoid re-reading events already processed. Only new events since the last poll timestamp are fetched.

---

## Monthly Cost Summary

At all realistic hotel staff scales, cf-admin costs **$0/month**:

```
Cloudflare Workers    $0  (well under 100K req/day free limit)
Cloudflare D1         $0  (well under 5M reads + 100K writes/day)
Cloudflare KV         $0  (well under 100K reads + 1K writes/day)
Cloudflare R2         $0  (shared with cf-astro; no admin-only cost)
Cloudflare Queues     $0  (admin is a producer only; minimal ops)
Supabase              $0  (shared with cf-astro; admin reads add negligible data)
Upstash Redis         $0  (shared; admin rate limit calls add ~50 commands/day)
─────────────────────────
cf-admin TOTAL        $0/month
```
