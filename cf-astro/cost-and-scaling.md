# cf-astro — Cost & Scaling

> Volume capacity, per-endpoint costs, and scaling projections.

---

## Current Cost: $0–$5/mo

All cf-astro operations fall within free tier limits at current scale.

| Resource | Current Usage | Free Tier | Utilization |
|----------|-------------|-----------|-------------|
| Workers requests | ~5K/mo | 10M/mo | <0.1% |
| D1 reads | ~500/day | 5M/day | <0.01% |
| KV reads (ISR) | ~200/day | 100K/day | 0.2% |
| KV writes (ISR) | ~50/day | 1K/day | 5% |
| R2 storage | ~300MB | 10GB | 3% |
| R2 reads | ~1K/mo | 10M/mo | <0.01% |
| Queue produces | ~100/mo | 1M/mo | <0.01% |
| Analytics Engine | ~200/day | 100K/day | 0.2% |

---

## Per-Endpoint Cost Analysis

### GET / (Homepage)

| Component | ISR HIT | ISR MISS |
|-----------|---------|----------|
| Workers CPU | ~0.5ms | ~15ms |
| KV read | 1 read ($0) | 1 read + 1 write ($0) |
| D1 (feature flags) | 0-1 read ($0) | 0-1 read ($0) |
| Supabase | 0 queries | 1-3 queries ($0) |
| Analytics Engine | 1 write ($0) | 1 write ($0) |
| **Total cost** | **~$0.000001** | **~$0.00001** |

### POST /api/booking

| Component | Cost per Request |
|-----------|-----------------|
| Workers CPU | ~20ms ($0) |
| Turnstile verify | 1 subrequest ($0) |
| Upstash rate limit | 2 Redis commands ($0) |
| Supabase INSERTs | 3-5 queries (~$0.000001) |
| Queue produces | 2 messages ($0) |
| Analytics Engine | 1 write ($0) |
| **Total cost** | **~$0.00001** |

At 1,000 bookings/month: **~$0.01** in compute costs.

---

## Scaling Projections

### At 10x Traffic (50K visits/mo)

| Resource | Usage | Cost |
|----------|-------|------|
| Workers | 50K req/mo | $0 (within Standard) |
| KV reads | 2K/day | $0 |
| KV writes | 500/day | $0 (approaching limit) |
| D1 reads | 5K/day | $0 |
| R2 storage | 1-2GB | $0 |
| Supabase queries | ~10K/mo | $0 |
| **TOTAL** | | **$0** (within free tiers) |

### At 100x Traffic (500K visits/mo)

| Resource | Usage | Cost |
|----------|-------|------|
| Workers | 500K req/mo | $0 (within Standard) |
| KV reads | 20K/day | $0 |
| KV writes | 5K/day | Needs Standard ($5/mo) |
| D1 reads | 50K/day | $0 |
| R2 storage | 5-10GB | $0 |
| Supabase queries | ~100K/mo | $0 |
| **TOTAL** | | **~$5/mo** |

### ISR Cache Hit Rate Impact

| Hit Rate | SSR Renders/Day (at 5K visits/day) | Supabase Cost |
|----------|-----------------------------------|--------------| 
| 50% | 2,500 | ~$0.001/day |
| 80% | 1,000 | ~$0.0004/day |
| 95% | 250 | ~$0.0001/day |

**Target**: 80%+ ISR cache hit rate to minimize Supabase queries.

---

## Capacity Limits

| Metric | Hard Limit | Current | Headroom |
|--------|-----------|---------|----------|
| Requests/day | 333K (Standard) | ~170 | 1,960x |
| KV reads/day | 100K (Free) | ~200 | 500x |
| KV writes/day | 1K (Free) | ~50 | 20x ⚠️ |
| D1 reads/day | 5M (Free) | ~500 | 10,000x |
| R2 reads/mo | 10M (Free) | ~1K | 10,000x |
| Queue ops/mo | 1M (Standard) | ~100 | 10,000x |

**First bottleneck**: KV writes at ~20x headroom. At 10x growth, may need KV Standard plan ($5/mo).
