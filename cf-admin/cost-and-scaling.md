# cf-admin — Cost & Scaling

Service-by-service cost breakdown, cron invocation counts, and upgrade triggers.

---

## Usage Profile

cf-admin has fundamentally different usage characteristics than the public cf-astro site:

| Dimension | cf-astro (public) | cf-admin |
|-----------|------------------|---------|
| Daily users | Hundreds of visitors | 2–5 concurrent admins max |
| Requests/day | ~2,000–5,000 | ~200–500 (plus 288 cron) |
| Write operations | Low (bookings only) | High (audit logs, CMS, user management) |
| CPU per request | ~2–5ms (ISR cache hits) | ~5–9ms (session + RBAC + PLAC) |
| Cron invocations | None | 288/day (every 5 minutes) |

---

## Monthly Cost Summary

At all realistic hotel staff scales, cf-admin costs **$0/month**:

```
Cloudflare Workers      $0    (well under 100K req/day free limit)
Cloudflare D1           $0    (well under 5M reads + 100K writes/day)
Cloudflare KV           $0    (well under 100K reads + 1K writes/day)
Cloudflare R2           $0    (images; $0 egress via CDN custom domain)
Cloudflare Queues       $0    (admin is producer only; very low volume)
Cloudflare Analytics    $0    (optional binding; low write volume)
Supabase PostgreSQL     $0    (shared with cf-astro; admin reads add negligible volume)
Upstash Redis           $0    (shared with other services; ~50 rate-limit commands/day)
Sentry                  $0    (shared DSN)
─────────────────────────────
cf-admin TOTAL          $0/month
```

---

## Service-by-Service Breakdown

### Cloudflare Workers

**Requests per day (estimated):**

| Source | Volume |
|--------|--------|
| Admin UI page loads (5 admins × 10–20 loads) | 50–100 |
| API routes (bookings, CMS, users, audit) | 100–200 |
| Cron trigger (`*/5 * * * *`) | 288 |
| Diagnostic health checks | ~20 |
| **Total** | **~460–610 requests/day** |

**Free tier limit:** 100,000 requests/day
**Headroom:** ~99.5% — cf-admin will never exhaust the Workers free tier at hotel staff scale.

**CPU per request:**
- KV session read (Layer 4): ~2–5ms
- PLAC hashmap lookup (Layer 8): <0.1ms
- Supabase role re-check (Layer 5, every 30 min, async): ~20–60ms (does not block response)
- Business logic: ~2–6ms
- **Total blocking CPU per request:** ~3–8ms (well under 10ms CPU time limit)

---

### Cloudflare KV (SESSION namespace)

**Reads per day:**

| Operation | Volume |
|-----------|--------|
| Session reads (one per request) | 460–610 |
| Revocation flag checks | 460–610 |
| **Total reads** | **~920–1,220/day** |
| Free tier limit | 100,000/day |
| Headroom | ~99% |

**Writes per day:**

| Operation | Volume |
|-----------|--------|
| New sessions (admin logins) | 5–15 |
| Role freshness patches (every 30 min per active admin) | 20–60 |
| PLAC map patches (every 60 min per active admin) | 10–30 |
| Revocation flag writes (rare) | 0–2 |
| Cron timestamp updates | 288 |
| **Total writes** | **~323–395/day** |
| Free tier limit | 1,000/day |
| Headroom | ~60–68% |

Note: The cron writes 1 KV key (`cf-audit-last-synced`) on each of its 288 daily invocations, which is the largest single contributor to write count. If write budget becomes constrained, cron frequency could be reduced.

---

### Cloudflare D1 (`madagascar-db`)

**Writes per day:**

| Source | Volume |
|--------|--------|
| Login forensics records | 5–15 |
| Ghost Audit log entries | 50–200 |
| CF Access audit log ingestion (cron) | 0–100 events/day |
| CMS content updates | 10–50 |
| Booking shadow state updates | 5–20 |
| Feature flag toggles | 0–5 |
| Diagnostic run results | 10–20 |
| **Total writes** | **~80–410/day** |
| Free tier limit | 100,000/day |
| Headroom | >99% |

**Reads per day:**

| Source | Volume |
|--------|--------|
| PLAC computation (1-hour TTL, ~5 active users) | 5–10 |
| Booking list/detail queries | 50–100 |
| User registry queries | 10–20 |
| Audit log viewer queries | 20–50 |
| Feature flag reads (D1 fallback only) | 5–20 |
| **Total reads** | **~90–200/day** |
| Free tier limit | 5,000,000/day |
| Headroom | >99.99% |

---

### Cron Trigger Analysis

The `*/5 * * * *` cron fires 288 times per day. Each invocation:

| Step | Cost |
|------|------|
| CF Access API call | 1 subrequest |
| New auth events to process | 0–20 per invocation (typically 0 at quiet periods) |
| D1 writes (new events) | 0–20 rows |
| KV write (timestamp update) | 1 write |
| Security alert emails (rare) | 0–1 per invocation |

Total cron contribution to daily Workers request count: 288 requests. At normal scale, combined admin + cron traffic stays well under 1,000 requests/day.

---

### Cloudflare R2 (`madagascar-images`)

- **Writes:** Only when admins upload images. At <10 image uploads/day: negligible.
- **Reads:** Served via `cdn.madagascarhotelags.com` custom domain. R2 egress via custom domain is $0.
- **Free tier:** 10GB storage, 1M Class A ops/month, 10M Class B ops/month.
- **Cost:** $0 at any realistic hotel image volume.

---

### Upstash Redis

Rate limiting operations per day:

| Endpoint | Limit | Calls/day (estimated) |
|----------|-------|----------------------|
| User management writes | 10/hour | ~20–50 |
| PLAC provisioning | 5/minute | ~10–20 |
| **Total Redis commands** | | **~30–70/day** |

Upstash free tier: 10,000 commands/day. Headroom: ~99%.

---

## Scaling Projections

| Scenario | Workers req/day | KV writes/day | D1 writes/day | Monthly Cost |
|----------|----------------|---------------|---------------|-------------|
| Current (2–5 admins) | ~600 | ~390 | ~300 | $0 |
| 10 admins | ~1,500 | ~700 | ~800 | $0 |
| 20 admins, heavy editing | ~5,000 | ~1,100 | ~2,000 | $0 (KV approaching limit) |
| 50 admins (hotel chain scale) | ~15,000 | ~3,000 | ~5,000 | Workers Std $5/mo; KV upgrade needed |

### First Bottleneck: KV Writes

The KV write free tier is 1,000 writes/day. The cron alone contributes 288/day. At ~20 actively editing admins, the remaining write budget could be exhausted.

**Upgrade path:** Workers Standard plan ($5/month) raises the KV write limit from 1,000/day to 1,000,000/month. This resolves all KV write constraints at any realistic hotel staff scale.

---

## Optimization Notes

**Session KV efficiency:** The session is read once per request (Layer 4) and passed via Astro `locals`. API route handlers do not perform additional KV reads.

**PLAC amortization:** PLAC access maps are computed once per 60 minutes per user (single D1 JOIN query), then served from the pre-computed `accessMap` in the KV session. All requests within the 60-minute window pay only the O(1) hashmap lookup cost.

**Ghost Audit Engine:** Audit log writes use `ctx.waitUntil()` — they never add to user-perceived latency. Even if D1 is temporarily slow, the admin's response arrives immediately.

**Cron cursor:** The CF Access audit poll uses the `cf-audit-last-synced` KV key as a cursor. Only events newer than the last poll are fetched — no duplicate processing.

**Role re-check async:** Supabase role freshness checks (~20–60ms) are dispatched via `ctx.waitUntil()` when the 30-minute window has elapsed. The current request is not blocked — the updated role is applied to the next request.
