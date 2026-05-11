# cf-astro — Cost & Scaling

Current cost breakdown, free tier headroom per binding, first bottleneck analysis, and scaling projections.

---

## Current Cost: $0/month

At 10–20 bookings per day (~300–600 bookings/month), all cf-astro operations fall well within free tier limits across every Cloudflare and third-party service used.

---

## Current Cost Breakdown

| Service | Current usage | Free tier limit | Utilization | Monthly cost |
|---------|-------------|----------------|-------------|-------------|
| Workers compute | <<1% of 100K req/day | 100,000 req/day (Standard) | <0.1% | $0 |
| KV reads (ISR hits) | Reads free | Unlimited reads | — | $0 |
| KV writes (ISR misses) | <<1K/day | 1,000 writes/day (Free) | <5% | $0 |
| D1 reads | <<5M/day | 5,000,000 reads/day | <0.01% | $0 |
| D1 writes | Minimal | 100,000 writes/day | <0.01% | $0 |
| R2 storage | <<10GB | 10 GB | <3% | $0 |
| R2 egress | — | $0 forever | — | $0 |
| R2 Class A ops (writes) | <<1M/month | 1,000,000/month | <0.01% | $0 |
| R2 Class B ops (reads) | <<10M/month | 10,000,000/month | <0.01% | $0 |
| Queue operations | <<10K ops/day | 1,000,000 ops/month | <0.01% | $0 |
| Analytics Engine | <<100K datapoints/day | 100,000/day | <0.2% | $0 |
| Workers AI | <<10K neurons/day | 10,000 neurons/day (rarely used) | Low | $0 |
| Sentry | Low error volume | Free tier (shared DSN) | Low | $0 |
| BetterStack (Logtail) | Low log volume | Free tier | Low | $0 |
| Upstash Redis | Low rate-limit checks | Free tier | Low | $0 |

**Total: $0/month**

---

## Free Tier Headroom Per Binding

### ISR_CACHE KV — First Bottleneck

KV writes are the most constrained resource. The free tier allows 1,000 writes/day.

| Scenario | KV writes/day | Headroom |
|----------|-------------|---------|
| Current (10–20 bookings/day) | ~50 | 20x |
| 5x growth (50–100 bookings/day) | ~250 | 4x |
| 10x growth (100–200 bookings/day) | ~500 | 2x |
| Upgrade trigger | ~1,000 | 1x (limit hit) |

**Why KV writes are the first bottleneck**: Every ISR cache MISS writes one KV entry per page. After a deployment, all pages are cold until visited. At 10x current traffic, writes approach the free limit.

**Upgrade path**: KV Standard plan — $5/month for unlimited writes.

### Workers Compute

| Scenario | Requests/day | Free limit | Headroom |
|----------|------------|-----------|---------|
| Current | ~170 | 100,000 | 588x |
| 10x growth | ~1,700 | 100,000 | 58x |
| 100x growth | ~17,000 | 100,000 | 5.8x |
| Upgrade trigger | ~100,000/day | — | Workers Paid: $5/month + $0.50/million |

### D1 Database

| Scenario | Reads/day | Free limit | Headroom |
|----------|---------|-----------|---------|
| Current | ~500 | 5,000,000 | 10,000x |
| 1000x growth | ~500,000 | 5,000,000 | 10x |

D1 is not a bottleneck at any foreseeable scale for this project.

### Analytics Engine

| Scenario | Datapoints/day | Free limit | Headroom |
|----------|-------------|-----------|---------|
| Current | ~200 | 100,000 | 500x |
| 10x growth | ~2,000 | 100,000 | 50x |
| 100x growth | ~20,000 | 100,000 | 5x |

### Workers AI

| Scenario | Neurons/day | Free limit | Headroom |
|----------|-----------|-----------|---------|
| Blog gen (rare) | ~5,000 per post | 10,000 | 2 posts/day max free |
| FAQ gen (rare) | ~1,000 per run | 10,000 | 10 runs/day max free |

Workers AI is used on-demand by admin only. At current usage (a few posts/month), it is nowhere near the limit.

---

## Email Costs (Resend) — First Paid Service

Resend charges based on email volume, not Cloudflare free tier.

| Plan | Emails/month | Cost |
|------|------------|------|
| Free | 3,000 | $0 |
| Pro | Unlimited | ~$20/month |

**Current email volume** (2 emails per booking: admin + customer, plus contact form):
- 10–20 bookings/day × 2 emails = 20–40 emails/day = 600–1,200/month
- Plus contact form emails (~10–30/month)
- Total: ~630–1,230/month — well within free 3,000/month

**Upgrade trigger**: ~45 bookings/day (1,350 emails/month from bookings alone approaches 3,000 limit). Resend is the first service that will require payment at scale.

---

## Scaling Projections

### 10x Current Scale (~100–200 bookings/day, ~50K visits/month)

| Resource | Usage | Cost |
|----------|-------|------|
| Workers compute | ~17K req/day | $0 (within 100K free) |
| KV reads | ~2K/day | $0 |
| KV writes | ~500/day | $0 (approaching 1K limit) |
| D1 reads | ~5K/day | $0 |
| R2 storage | ~1–2 GB | $0 |
| Queue ops | ~10K/day | $0 |
| Analytics Engine | ~2K datapoints/day | $0 |
| Resend emails | ~12K–24K/month | $0 (upgrade needed — Pro plan) |
| **TOTAL** | | **~$0–$20/month** (Resend Pro if bookings exceed 1,500/month) |

### 100x Current Scale (~1,000–2,000 bookings/day, ~500K visits/month)

| Resource | Usage | Cost |
|----------|-------|------|
| Workers compute | ~170K req/day | $5/month (Workers Paid, $0.50/million) |
| KV reads | ~20K/day | $0 |
| KV writes | ~5K/day | $5/month (KV Standard) |
| D1 reads | ~50K/day | $0 |
| R2 storage | ~5–10 GB | $0 |
| Queue ops | ~100K/day | $0 |
| Analytics Engine | ~20K datapoints/day | $0 |
| Resend emails | ~120K–240K/month | ~$20–$50/month (Resend Pro) |
| **TOTAL** | | **~$30–$60/month** |

---

## ISR Cache Impact on Cost

Higher ISR cache hit rates directly reduce Supabase query load and Workers CPU time.

| ISR hit rate | SSR renders/day (at 5K visits/day) | Supabase queries saved |
|-------------|-----------------------------------|----------------------|
| 50% | 2,500 renders | 50% |
| 80% | 1,000 renders | 80% |
| 95% | 250 renders | 95% |

The 24-hour TTL and deploy-scoped cache keys mean a fresh deployment temporarily drops hit rate to 0% until pages are warmed. At 5K visits/day, cache warms quickly (<1 hour).

---

## Upgrade Triggers Summary

| Trigger condition | Service to upgrade | Estimated cost |
|------------------|--------------------|---------------|
| KV writes approach 1K/day (~10x traffic) | KV Standard | $5/month |
| Workers requests exceed 100K/day (~100x traffic) | Workers Paid | $5/month base |
| Resend emails exceed 3K/month (~45 bookings/day) | Resend Pro | ~$20/month |
| Upstash Redis rate limit hits (very high scale) | Upstash Pay-as-you-go | Minimal |

**Bottom line**: At current scale (10–20 bookings/day), total infrastructure cost is $0/month. The first dollar spent will be on Resend at approximately 45 bookings/day sustained. The first Cloudflare cost appears at approximately 10x current traffic for KV writes.
