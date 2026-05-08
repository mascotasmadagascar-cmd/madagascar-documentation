# 07 — Scalability & Limits

> Hard limits, growth projections, and horizontal scaling strategies.

---

## Cloudflare Workers Hard Limits

### Compute

| Limit | Free | Standard | Enterprise |
|-------|------|----------|-----------|
| Requests | 100K/day | 10M/mo included | Unlimited |
| CPU time / invocation | 10ms | 30s | 30s+ |
| Memory / isolate | 128MB | 128MB | 256MB |
| Script size | 1MB | 10MB | 10MB+ |
| Environment variables | 64 | 128 | 128+ |
| Subrequest limit | 50/req | 1,000/req | 1,000+/req |
| Request body | 100MB | 100MB | 500MB |
| Response body | Streaming | Streaming | Streaming |
| Cron triggers | 3 | 5 | 5+ |
| Worker scripts / account | 30 | 500 | Unlimited |

### D1

| Limit | Free | Standard |
|-------|------|----------|
| Rows read / day | 5M | 25B/mo |
| Rows written / day | 100K | 50M/mo |
| Storage | 5GB per DB | 10GB incl. ($0.75/GB after) |
| Databases / account | 50K | 50K |
| Max DB size | 10GB | 10GB |
| Max rows / table | No limit | No limit |
| Query size | 100KB | 100KB |
| Batch operations | 10 statements | 10 statements |
| Time Travel | 30 days | 30 days |

### KV

| Limit | Free | Standard |
|-------|------|----------|
| Reads / day | 100K | 10M/mo |
| Writes / day | 1K | 1M/mo |
| Deletes / day | 1K | 1M/mo |
| List ops / day | 1K | 1M/mo |
| Storage | 1GB | 1GB ($0.50/GB after) |
| Namespaces / account | 200 | 200 |
| Key size | 512 bytes | 512 bytes |
| Value size | 25MB | 25MB |
| Metadata / key | 1KB | 1KB |

### R2

| Limit | Free | Standard |
|-------|------|----------|
| Storage | 10GB | $0.015/GB/mo |
| Class A ops (write) | 1M/mo | $4.50/M |
| Class B ops (read) | 10M/mo | $0.36/M |
| Egress | **$0 always** | **$0 always** |
| Max object size | 5TB | 5TB |
| Buckets / account | 1,000 | 1,000 |

### Queues

| Limit | Free (Standard plan) | Paid |
|-------|---------------------|------|
| Operations | 1M/mo | $0.40/M after |
| Message size | 128KB | 128KB |
| Batch size | 100 messages | 100 messages |
| Retention | 4 days | 4 days |
| Max delay | 12 hours | 12 hours |
| Queues / account | 100 | 100 |
| Consumers / queue | 20 | 20 |

### Vectorize

| Limit | Free | Paid |
|-------|------|------|
| Vectors stored | 200K | 5M ($5/mo) |
| Dimensions | 1536 max | 1536 max |
| Queries / mo | 30M | 50M |
| Metadata / vector | 10KB | 10KB |
| Namespaces / index | 1K | 1K |

### Workers AI

| Limit | Free | Paid |
|-------|------|------|
| Neurons / day | 10K | Unlimited ($0.011/1K) |

---

## Growth Projections

### Current State: "Startup" (Month 1-6)

```
Daily traffic:     ~100-500 visitors
Bookings/month:    ~10-50
Chat conversations: ~5-20/day
Admin users:        2-5
Emails/month:       ~100-500
```

**Resource utilization**: <5% of all free tiers.

### Phase 2: "Growth" (10x — Month 6-18)

```
Daily traffic:     ~1,000-5,000 visitors
Bookings/month:    ~100-500
Chat conversations: ~50-200/day
Admin users:        5-15
Emails/month:       ~1,000-5,000
```

| Resource | Usage | Limit | Utilization |
|----------|-------|-------|-------------|
| Workers requests | 150K/mo | 10M/mo | 1.5% |
| D1 reads | 60K/day | 5M/day | 1.2% |
| D1 writes | 6K/day | 100K/day | 6% |
| KV reads | 5K/day | 100K/day | 5% |
| KV writes | 500/day | 1K/day | 50% ⚠️ |
| R2 storage | 2GB | 10GB | 20% |
| Queue ops | 15K/mo | 1M/mo | 1.5% |
| Resend emails | 150/day | 100/day | 150% ⛔ |

**Action Required**:
- KV writes approaching free limit → upgrade to Standard ($5/mo) or optimize write patterns
- Resend exceeds 100/day → upgrade to Pro ($20/mo)

### Phase 3: "Scale" (100x — Month 18-36)

```
Daily traffic:     ~10,000-50,000 visitors
Bookings/month:    ~1,000-5,000
Chat conversations: ~500-2,000/day
Admin users:        15-50
Emails/month:       ~10,000-50,000
```

| Resource | Usage | Plan Needed | Monthly Cost |
|----------|-------|-------------|-------------|
| Workers | 5M req/mo | Standard | $5 + $0 overage |
| D1 | 600K reads/day, 60K writes/day | Standard | $5 |
| KV | 50K reads/day, 5K writes/day | Standard | $5 |
| R2 | 20–50GB | Free + overage | $0.15–0.60 |
| Queues | 150K ops/mo | Standard (included) | $0 |
| Supabase | 5GB data | Pro | $25 |
| Upstash | 50K cmds/day | Pay-as-you-go | $3 |
| Resend | 1,500/day | Pro | $20 |
| Sentry | 10K errors/mo | Team | $26 |
| LLM APIs | 2K convos/day | Pay-as-you-go | $30–60 |

**Total at 100x**: ~$120–150/mo

---

## Horizontal Scaling Strategies

### 1. Database Read Replicas
When Supabase free tier PostgreSQL becomes a bottleneck:
- **Option A**: Supabase Pro with read replicas ($25/mo + $100/mo per replica)
- **Option B**: Migrate read-heavy queries to D1 (cached at edge, $0)
- **Option C**: Use Cloudflare Hyperdrive for connection pooling

### 2. Multi-Region Supabase
Currently single-region (us-east-1). For true global scale:
```
cf-astro (MX user) → CF Mexico City PoP → D1 (edge, <1ms)
                                         → Supabase (us-east-1, ~30ms)
```
Mitigation: Cache Supabase responses in D1 or KV for hot paths.

### 3. Queue Scaling
Cloudflare Queues auto-scale consumers. If email volume exceeds Resend rate limits:
- Add a second Queue consumer with delayed processing
- Implement priority queuing (urgent emails first)
- Use Resend batch API for bulk sends

### 4. Chatbot AI Scaling
LLM API rate limits are the real bottleneck:
- **Gemini**: 1,500 RPM (Flash), 360 RPM (Pro)
- **Claude**: 4,000 RPM (Sonnet)
- **Workers AI**: No rate limit (free tier has neuron budget)

Mitigation:
- Upstash response cache reduces duplicate queries by 20–40%
- Static shortcut detection handles 15–20% of messages without LLM
- Workers AI as final fallback (unlimited, $0, lower quality)

### 5. ISR Cache Optimization
Current ISR strategy is per-deployment hash. At scale:
- Increase KV cache TTL from 24h to 7d for stable pages
- Use stale-while-revalidate pattern
- Deploy-scoped keys auto-clean on new deployments

---

## Bottleneck Analysis

| Scale | Primary Bottleneck | Mitigation |
|-------|-------------------|-----------|
| 1x (current) | None | All within free tiers |
| 10x | KV writes, Resend limits | Upgrade Resend to Pro, optimize KV writes |
| 100x | Supabase connections, LLM rate limits | Hyperdrive, response caching, model diversity |
| 1000x | Supabase Pro limits, Sentry volume | Read replicas, custom logging pipeline |
| 10000x | Cloudflare Enterprise needed | Enterprise plan, dedicated infrastructure |
