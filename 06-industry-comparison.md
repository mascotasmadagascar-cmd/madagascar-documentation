# 06 — Industry Comparison

> How the Madagascar edge stack compares to industry-standard approaches.

---

## Platform Comparison Matrix

| Feature | Madagascar (CF Workers) | AWS (Lambda + S3 + RDS) | Vercel (Edge + Serverless) | Railway / Render | Fly.io |
|---------|------------------------|------------------------|---------------------------|-----------------|--------|
| **Cold Start** | <5ms | 200–1500ms | 10–2500ms | 50–500ms (always-on) | 300–3000ms |
| **Global PoPs** | 330+ | ~30 regions | ~20 edge locations | ~6 regions | ~35 regions |
| **Egress Cost** | $0 | $0.09/GB | $0.15/GB | $0.10/GB | $0.02/GB |
| **Database at Edge** | D1 (SQLite, co-located) | None (DynamoDB regional) | None | N/A | LiteFS (SQLite) |
| **KV Store** | KV (built-in, $0) | ElastiCache ($15+/mo) | KV ($0 basic) | Redis ($7+/mo) | None built-in |
| **Object Storage** | R2 ($0 egress) | S3 ($0.09/GB egress) | Blob ($0 to $5) | N/A | Volumes (local) |
| **Queue System** | Queues (built-in, $0) | SQS ($0.40/M) | None | N/A | None built-in |
| **Vector DB** | Vectorize (built-in) | OpenSearch ($50+/mo) | None | N/A | None built-in |
| **AI Inference** | Workers AI ($0 free tier) | SageMaker ($50+/mo) | None | N/A | GPU ($1.2/hr) |
| **Auth at Edge** | Zero Trust (built-in) | Cognito (complex) | NextAuth | Custom | Custom |
| **DDoS Protection** | Included (296Tbps) | AWS Shield ($3K/mo) | Included (basic) | Limited | Limited |
| **Min Monthly Cost** | $5 (Standard) | ~$30+ | $20 (Pro) | $5+ | $5+ |

---

## Architecture Pattern Comparison

### 1. Madagascar: Edge-Native Microservices

```
User → CF Edge (330 PoPs) → V8 Isolate → D1/KV/R2 (edge) → Supabase (regional)
```

**Strengths**:
- Compute + storage at edge = minimal latency
- All CF primitives communicate via bindings ($0 cost, 0 network hop)
- Horizontal scale is automatic and instant

**Weaknesses**:
- 128MB memory limit per isolate
- No persistent filesystem
- V8 only (no Python/Go/Rust — though WASM is supported)
- D1 has SQLite limitations (no stored procedures, no LISTEN/NOTIFY)

### 2. Traditional: Regional Monolith (Node.js + PostgreSQL)

```
User → CDN → Load Balancer → EC2/ECS → PostgreSQL
```

**Strengths**:
- Unlimited memory and CPU
- Full PostgreSQL features (CTEs, window functions, extensions)
- Mature ecosystem

**Weaknesses**:
- Single region (or expensive multi-region)
- Manual scaling, ops overhead
- $50–200/mo minimum for small apps

### 3. Modern: Vercel + Neon (Serverless)

```
User → Vercel Edge → Serverless Function (us-east-1) → Neon PostgreSQL
```

**Strengths**:
- Best DX for Next.js teams
- Excellent preview deployments
- Neon serverless Postgres is innovative

**Weaknesses**:
- Edge functions are limited (no database access from edge in most cases)
- Serverless functions are regional (cold starts)
- Expensive at scale ($20/mo Pro minimum, $0.15/GB egress)
- No built-in queues, KV limited

### 4. Container-Based: Railway / Render

```
User → CDN → Container → PostgreSQL
```

**Strengths**:
- Full runtime flexibility
- Persistent processes (WebSockets native)
- Simple mental model

**Weaknesses**:
- Always-on cost ($5–20/mo per service)
- Regional deployment
- Manual CDN configuration
- No edge compute

---

## Cost-Efficiency Ranking

At different scales (monthly):

### Small Scale (10K req/mo)

| Platform | Monthly Cost | Notes |
|----------|-------------|-------|
| **1. Madagascar (CF Workers)** | **$5** | Standard plan covers everything |
| 2. Vercel Hobby + Supabase Free | $0 | Limited to hobby (no commercial) |
| 3. Railway | $5–10 | Single container + managed Postgres |
| 4. AWS Lambda + RDS | $30+ | RDS minimum is ~$15/mo |
| 5. Fly.io | $5–15 | micro-VM + managed Postgres |

### Medium Scale (500K req/mo)

| Platform | Monthly Cost | Notes |
|----------|-------------|-------|
| **1. Madagascar (CF Workers)** | **$10–40** | Free tier covers most infra |
| 2. Railway | $25–50 | CPU/memory scaling |
| 3. Vercel Pro + Neon | $45–80 | $20 base + function invocations |
| 4. Fly.io | $30–60 | Multi-region adds cost |
| 5. AWS Lambda + Aurora | $80–150 | Aurora Serverless v2 minimum |

### Large Scale (5M req/mo)

| Platform | Monthly Cost | Notes |
|----------|-------------|-------|
| **1. Madagascar (CF Workers)** | **$80–160** | Supabase Pro is largest cost |
| 2. Railway | $100–200 | Dedicated resources needed |
| 3. Fly.io | $120–250 | Multi-region with Postgres |
| 4. Vercel Enterprise + Neon | $200–400 | Enterprise plan + egress costs |
| 5. AWS Full Stack | $270–800 | RDS + Lambda + S3 + CloudFront + SQS |

---

## Feature-by-Feature Deep Comparison

### Database Strategy

| Approach | Madagascar | AWS Equivalent | Vercel Equivalent |
|----------|-----------|---------------|------------------|
| Primary DB | Supabase PostgreSQL | Amazon RDS/Aurora | Neon / PlanetScale |
| Edge Cache DB | Cloudflare D1 | DynamoDB Global Tables | None (Redis?) |
| Session Store | Cloudflare KV | ElastiCache Redis | Vercel KV |
| Object Store | Cloudflare R2 | Amazon S3 | Vercel Blob |
| Vector Store | Cloudflare Vectorize | OpenSearch / Pinecone | Pinecone / Weaviate |

**Madagascar Advantage**: D1 at the edge eliminates the "database is in us-east-1" problem. Feature flags, audit logs, and KB lookups happen at <1ms latency, regardless of user location.

### AI/ML Integration

| Feature | Madagascar | AWS | Vercel |
|---------|-----------|-----|--------|
| Edge inference | Workers AI (free 10K neurons/day) | SageMaker Serverless ($50+/mo) | None |
| Vector search | Vectorize (free 200K vectors) | OpenSearch ($50+/mo) | None |
| Embedding generation | Workers AI (free) | Bedrock ($) | OpenAI ($) |
| LLM integration | Direct API (Gemini/Claude) | Bedrock | AI SDK |

**Madagascar Advantage**: Workers AI + Vectorize provide a complete RAG pipeline at $0, with inference happening at the edge (same PoP as the Worker). AWS requires provisioned capacity; Vercel has no built-in AI infrastructure.

### Security

| Feature | Madagascar | AWS | Vercel |
|---------|-----------|-----|--------|
| DDoS | Included (296 Tbps) | AWS Shield Standard (basic) or $3K/mo (Advanced) | Included (basic) |
| WAF | Included (basic rules) | AWS WAF ($5/mo + $1/M req) | None |
| Auth gateway | Zero Trust ($0 for <50 users) | Cognito (complex setup) | NextAuth (DIY) |
| CAPTCHA | Turnstile ($0, privacy-first) | None built-in | None built-in |
| Rate limiting | Upstash ($0 free tier) | API Gateway throttling | None built-in |

**Madagascar Advantage**: Enterprise-grade security (DDoS, WAF, Zero Trust, CAPTCHA) included at $0. AWS charges $3K+/mo for equivalent Shield Advanced + WAF. Vercel offers basic protection only.

---

## When NOT to Choose Cloudflare Workers

Workers are not ideal for every use case:

| Scenario | Better Alternative | Why |
|----------|-------------------|-----|
| Long-running compute (>30s) | AWS Lambda (15min) or ECS | Workers time out at 30s |
| Heavy memory usage (>128MB) | Container (Railway, Fly.io) | Isolate memory cap |
| Binary dependencies (sharp, puppeteer) | Container or Lambda layer | No native binaries in V8 |
| Complex SQL (stored procs, extensions) | Dedicated PostgreSQL | D1 is SQLite, limited |
| WebSocket-heavy apps | Durable Objects or Fly.io | Standard Workers don't hold state |
| ML model training | AWS SageMaker, GPU instances | Workers AI is inference-only |

---

## Industry Standards Compliance

| Standard | Madagascar Status | Notes |
|----------|------------------|-------|
| SOC 2 Type II | ✅ Via Cloudflare | Cloudflare's infrastructure is SOC 2 certified |
| GDPR | ✅ Partial | Data minimization, IP hashing, consent collection |
| LFPDPPP (Mexican) | ✅ Full | ARCO rights implementation, consent records |
| PCI DSS | ⚠️ N/A | No payment processing (delegated to Stripe/PayPal) |
| OWASP Top 10 | ✅ Addressed | All 10 categories mitigated (see Security Architecture) |
| ISO 27001 | ✅ Via Cloudflare | Cloudflare's infrastructure is ISO 27001 certified |
