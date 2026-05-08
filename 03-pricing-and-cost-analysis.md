# 03 — Pricing & Cost Analysis

> Detailed cost breakdown at current scale and projections at 10x/100x growth.

---

## Executive Summary

| Metric | Current (Free Tier) | 10x Growth | 100x Growth |
|--------|-------------------|------------|-------------|
| **Monthly Cost** | **$0–$5** | **~$22–35/mo** | **~$65–120/mo** |
| **Requests/mo** | ~10K–50K | ~500K | ~5M |
| **Equivalent AWS Cost** | ~$15–30/mo | ~$80–150/mo | ~$350–800/mo |
| **Cost Advantage** | 3–6x cheaper | 3–5x cheaper | 4–7x cheaper |

---

## Service-by-Service Breakdown

### 1. Cloudflare Workers (All 4 Services)

| Tier | Requests/day | CPU Time | Cost |
|------|-------------|----------|------|
| **Free** | 100,000 req/day | 10ms/invocation | **$0** |
| **Standard** (current) | 10M req/mo | 30M CPU ms/mo | **$5/mo** base |
| **Overage** | +$0.30/M requests | +$0.02/M CPU ms | Per usage |

**Current Usage**: ~1K–5K requests/day across all 4 workers → **well within free tier limits**.

```
At 10x (50K req/day = 1.5M/mo):
  Workers cost = $5 base + $0 overage = $5/mo

At 100x (500K req/day = 15M/mo):  
  Workers cost = $5 base + (5M × $0.30/M) = $6.50/mo
```

### 2. Cloudflare D1

| Tier | Reads/day | Writes/day | Storage | Cost |
|------|----------|------------|---------|------|
| **Free** | 5M reads/day | 100K writes/day | 5GB | **$0** |
| **Standard** | 25B reads/mo | 50M writes/mo | 10GB incl. | **$5/mo** |

**Current Usage**: 2 databases (`madagascar-db`, `chatbot-kb`) — ~2K reads/day, ~200 writes/day → **<1% of free tier**.

```
At 100x: 200K reads/day, 20K writes/day → still within free tier
```

### 3. Cloudflare KV

| Tier | Reads/day | Writes/day | Storage | Cost |
|------|----------|------------|---------|------|
| **Free** | 100K reads/day | 1K writes/day | 1GB | **$0** |
| **Standard** | 10M reads/mo | 1M writes/mo | 1GB incl. | **$5/mo** |

**Current Usage**: 2 namespaces (`ISR_CACHE`, `SESSION`)
- Sessions: ~5–20 reads/day, ~2–5 writes/day
- ISR Cache: ~50–200 reads/day, ~10–50 writes/day
- **Total**: <300 reads/day → **<1% of free tier**

```
At 100x: ~30K reads/day → 30% of free tier (still free)
```

### 4. Cloudflare R2

| Tier | Storage | Class A ops | Class B ops | Egress | Cost |
|------|---------|-------------|-------------|--------|------|
| **Free** | 10GB | 1M/mo | 10M/mo | **$0 always** | **$0** |
| **Paid** | $0.015/GB | $4.50/M | $0.36/M | **$0 always** | Per usage |

**Current Usage**: 2 buckets (`madagascar-images`, `arco-documents`) — ~500MB total → **5% of free tier**.

Key Advantage: **$0 egress forever** (vs S3's $0.09/GB).

```
At 100x with 50GB images:
  R2 = (50 - 10) × $0.015 = $0.60/mo
  
Equivalent S3 at 50GB + 500GB egress/mo:
  S3 = $1.15 + $45.00 = $46.15/mo ← 77x more expensive
```

### 5. Cloudflare Queues

| Tier | Operations/mo | Messages | Cost |
|------|--------------|----------|------|
| **Standard** | 1M ops/mo | Included | **$0** (included in Workers Standard) |
| **Overage** | +$0.40/M ops | — | Per usage |

**Current Usage**: ~5–20 emails/day = ~150–600 ops/mo → **<0.1% of free tier**.

### 6. Cloudflare Workers AI

| Tier | Neurons/day | Cost |
|------|------------|------|
| **Free** | 10,000 neurons/day | **$0** |
| **Paid** | Unlimited | $0.011/1K neurons |

**Current Usage**: Used for embedding generation during KB admin operations. Sporadic usage, ~100–500 neurons/day → **<5% of free tier**.

### 7. Cloudflare Vectorize

| Tier | Stored vectors | Queries/mo | Cost |
|------|---------------|------------|------|
| **Free** | 200K vectors | 30M queries/mo | **$0** |
| **Paid** | 5M vectors | 50M queries/mo | **$5/mo** |

**Current Usage**: KB entries → ~50–200 vectors → **<0.1% of free limit**.

### 8. Cloudflare Analytics Engine

| Tier | Data points/day | Cost |
|------|----------------|------|
| **Free** | 100K/day | **$0** (included in Workers Standard) |

---

## External Services

### 9. Supabase

| Resource | Free Tier | Pro Plan ($25/mo) |
|----------|-----------|-------------------|
| Database | 500MB, 2 projects | 8GB, unlimited |
| Auth | 50K MAU | 100K MAU |
| Bandwidth | 5GB | 250GB |
| Edge Functions | 500K invocations | 2M invocations |
| Realtime | 200 concurrent | 500 concurrent |

**Current Usage**: 1 project, ~50MB data, ~100 MAU admin → **10% of free tier**.

```
At 100x:
  Data growth to ~5GB → requires Pro plan ($25/mo)
  But: only the database portion scales; auth/realtime still low.
  
Realistic cost at 100x: $25/mo
```

### 10. Upstash Redis

| Tier | Commands/day | Data | Cost |
|------|-------------|------|------|
| **Free** | 10K commands/day | 256MB | **$0** |
| **Pay-as-you-go** | Unlimited | Unlimited | $0.2/100K commands |

**Current Usage**: Rate limiting (~50–200 cmds/day) + chatbot cache (~10–50 cmds/day) → **~2% of free tier**.

```
At 100x: ~25K commands/day → may need Pay-as-you-go
  Cost: ~25K × 30 = 750K cmds/mo × $0.2/100K = $1.50/mo
```

### 11. Resend

| Tier | Emails/mo | Domains | Cost |
|------|----------|---------|------|
| **Free** | 100/day (3,000/mo) | 1 | **$0** |
| **Pro** | 50K/mo | Unlimited | **$20/mo** |

**Current Usage**: ~5–20 emails/day → **5–20% of free tier**.

```
At 10x: ~50–200 emails/day → may exceed free tier
  Cost: $20/mo (Pro)

At 100x: ~500–2000 emails/day = ~15K–60K/mo
  Cost: $20/mo (Pro) or $40/mo (Business)
```

### 12. Sentry

| Tier | Errors/mo | Performance events | Cost |
|------|----------|-------------------|------|
| **Developer** | 5K | 10K transactions | **$0** |
| **Team** | 50K | 100K transactions | **$26/mo** |

**Current Usage**: 3 workers integrated, ~100 errors/mo, ~1K transactions → **2% of free tier**.

---

## Total Cost Summary

### Current Scale (~10K req/mo)
```
Cloudflare Workers    $5.00 (Standard plan, covers all 4 workers)
Cloudflare D1         $0.00
Cloudflare KV         $0.00
Cloudflare R2         $0.00
Cloudflare Queues     $0.00
Cloudflare AI         $0.00
Cloudflare Vectorize  $0.00
Supabase              $0.00
Upstash               $0.00
Resend                $0.00
Sentry                $0.00
─────────────────────────────
TOTAL                 $5.00/month
```

### At 10x Growth (~500K req/mo)
```
Cloudflare Workers    $5.00
Cloudflare D1         $0.00
Cloudflare KV         $0.00
Cloudflare R2         $0.00
Cloudflare Queues     $0.00
Supabase              $0.00
Upstash               $0.00
Resend                $0.00–$20.00 (may need Pro)
Sentry                $0.00
LLM APIs              $5.00–$15.00 (Gemini/Claude usage)
─────────────────────────────
TOTAL                 $10.00–$40.00/month
```

### At 100x Growth (~5M req/mo)
```
Cloudflare Workers    $6.50
Cloudflare D1         $0.00
Cloudflare KV         $0.00
Cloudflare R2         $0.60
Cloudflare Queues     $0.00
Supabase              $25.00 (Pro plan)
Upstash               $1.50
Resend                $20.00–$40.00
Sentry                $26.00
LLM APIs              $20.00–$60.00
─────────────────────────────
TOTAL                 $99.60–$159.60/month
```

---

## Comparative Analysis: Same App on AWS / Vercel

### At 100x Scale (5M req/mo)

| Cost Category | Madagascar (CF Edge) | AWS Lambda + Aurora | Vercel + Neon |
|---------------|---------------------|--------------------|--------------| 
| Compute | $6.50 | $45.00 | $20.00 |
| Database | $25.00 (Supabase) | $80.00 (Aurora) | $25.00 (Neon) |
| Storage | $0.60 | $46.00 (S3 + egress) | $5.00 |
| Email | $20–40 | $5.00 (SES) | $20–40 (Resend) |
| Queue/Async | $0.00 | $15.00 (SQS + Lambda) | N/A |
| CDN/Edge | $0.00 (included) | $35.00 (CloudFront) | $0.00 (included) |
| Observability | $26.00 | $40.00 (CloudWatch) | $26.00 |
| **TOTAL** | **$78–$99** | **$266–$286** | **$96–$116** |

**Madagascar's edge stack is 2.5–3x cheaper than AWS** at 100x scale, primarily due to:
1. $0 egress (R2 vs S3)
2. Included CDN (vs CloudFront)
3. Free tier breadth (D1, KV, Queues, Vectorize all free)
4. No NAT gateway / VPC costs

---

## LLM API Cost Deep-Dive

### Per-Conversation Cost Estimate

| Model | Input $/M tokens | Output $/M tokens | Avg Conversation (5 turns) | Cost/Conversation |
|-------|------------------|-------------------|---------------------------|-------------------|
| Gemini 2.5 Flash | $0.15 | $0.60 | ~2K in + 1.5K out | **$0.001** |
| Gemini 2.5 Pro | $1.25 | $10.00 | ~2K in + 1.5K out | **$0.018** |
| Claude 3.5 Sonnet | $3.00 | $15.00 | ~2K in + 1.5K out | **$0.029** |
| Claude Sonnet 4.6 | $3.00 | $15.00 | ~2K in + 1.5K out | **$0.029** |
| Workers AI (Llama) | Free | Free | ~2K in + 1.5K out | **$0.000** |

With the current Gemini Flash primary → Claude fallback chain:
- **95% of conversations**: $0.001 (Gemini Flash)
- **5% fallback**: $0.029 (Claude)
- **Weighted average**: ~$0.002/conversation

```
At 1,000 conversations/mo: ~$2.00 in LLM costs
At 10,000 conversations/mo: ~$20.00 in LLM costs
```

The Upstash response cache further reduces this by 20–40% through dedup of repeated questions.
