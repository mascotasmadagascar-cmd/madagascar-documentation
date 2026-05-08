# 13 — Free Tier Master Reference

> Single source of truth for every service limit, current usage estimate, and upgrade trigger.
> **Last verified**: May 2026

---

## How to Read This Document

- **Free Limit**: The hard ceiling before charges begin
- **Est. Daily Usage**: Rough estimate at current hotel volume (~50 visitors/day)
- **Headroom**: How far below the limit we are (`99%` = almost nothing used)
- **First Hit**: When this service would require an upgrade (what visitor/booking volume triggers it)

---

## Cloudflare Services

### Workers (Compute — All 4 Services)

| Metric | Free Limit | Est. Daily Usage | Headroom | Notes |
|--------|-----------|-----------------|----------|-------|
| Requests/day | 100,000 | ~2,000–5,000 | ~95–98% | All 4 workers share this pool |
| CPU time per request | 10ms | ~2–5ms | ~50–80% | Middleware-heavy requests are ~8ms |
| Memory per Worker | 128 MB | ~30–60 MB | ~50–75% | |
| Worker script size | 3 MB | ~400–800 KB each | ~75% | |
| Cron triggers/account | 5 | 2 (cf-admin: 5min poll; cf-chatbot: 2AM cleanup) | 60% | |

**First bottleneck**: At ~50,000 requests/day (roughly 1,500 visitors/day at ~33 requests/visit). Upgrade to Workers Standard ($5/mo) adds 10M requests/month.

---

### D1 SQLite Database

Two separate databases: `madagascar-db` (cf-astro + cf-admin) and `chatbot-kb` (cf-chatbot).

| Metric | Free Limit | Est. Daily Usage | Headroom |
|--------|-----------|-----------------|----------|
| Rows read/day | 5,000,000 | ~5,000–20,000 | ~99.6% |
| Rows written/day | 100,000 | ~200–500 | ~99.8% |
| Storage total | 5 GB | ~50–100 MB | ~98% |
| Max databases | 10 | 2 | 80% free |
| Queries per invocation | 50 | ~3–8 | ~84% |

**Tables in `madagascar-db`**: `admin_feature_flags`, `admin_pages`, `admin_login_forensics`, `cms_content`, and more (cf-admin owned)
**Tables in `chatbot-kb`**: `kb_entries`, `kb_entries_fts` (FTS5 virtual), `bot_config`, `bot_conversations`

**First bottleneck**: D1 is unlikely to be the bottleneck at any realistic hotel scale. 100x current volume still fits within free tier.

---

### KV (Key-Value Store)

Three namespaces in use: `SESSION` (cf-admin), `cf-astro-session`, `ISR_CACHE` (cf-astro).

| Metric | Free Limit | Est. Daily Usage | Headroom |
|--------|-----------|-----------------|----------|
| Reads/day | 100,000 | ~2,000–5,000 | ~95–98% |
| Writes/day | **1,000** | ~50–200 | ~80–95% |
| Deletes/day | 1,000 | ~10–50 | ~95–99% |
| List requests/day | 1,000 | ~5–20 | ~98% |
| Storage/account | 1 GB | ~10–50 MB | ~95–99% |

> ⚠️ **KV writes are the tightest free tier constraint** in the system. At 10x growth, ISR revalidation events (cache purges) could approach the 1,000 writes/day limit. Monitor this metric first.

**Write sources**:
- cf-admin sessions: 1 write per admin login (rare)
- ISR cache writes: 1 write per unique URL per deployment (bounded by site pages)
- ISR revalidation: 1 delete + 1 write per invalidated URL per content edit

**First bottleneck**: ~100 content edits/day OR ~500 simultaneous admin users. Well below realistic levels.

---

### R2 Object Storage

Two buckets: `madagascar-images` (public gallery + CMS) and `arco-documents` (private legal docs).

| Metric | Free Limit | Est. Monthly Usage | Headroom |
|--------|-----------|-------------------|----------|
| Storage | 10 GB | ~500 MB–2 GB | ~80–95% |
| Class A operations (writes) | 1,000,000/month | ~200–500 | ~99.9% |
| Class B operations (reads) | 10,000,000/month | ~50,000–200,000 | ~98–99.5% |
| Egress | **$0 always** | — | — (unlimited) |

**Key advantage**: R2 egress is permanently free. Equivalent storage on AWS S3 with the same traffic would cost ~$45/month in data transfer alone.

**First bottleneck**: Storage at 10 GB. Reached if ~500–1,000 high-res photos are uploaded uncropped. Solution: compress images before upload (already recommended in RULES).

---

### Queues (Email Delivery)

One queue: `madagascar-emails` (producer: cf-astro + cf-admin; consumer: cf-email-consumer).

| Metric | Free Limit | Est. Daily Usage | Headroom |
|--------|-----------|-----------------|----------|
| Operations/day | 10,000 | ~30–100 | ~99% |
| Message retention | 24 hours | — | Sufficient |
| Max queues | 10,000 | 1 (+ 1 DLQ planned) | 99.9% |

> ⚠️ **24-hour retention** on the free plan means unprocessed messages are dropped after 24 hours. If cf-email-consumer is down for more than 24 hours, queued emails are lost. Paid plan offers 14-day retention. See [Implementations & Blockers](./implementations-and-blockers.md) for DLQ setup.

**First bottleneck**: 10,000 operations/day — hit only at ~5,000 emails/day. Not a realistic constraint.

---

### Workers AI

Used by cf-chatbot as the last-resort fallback LLM and by cf-astro for optional blog/FAQ generation.

| Metric | Free Limit | Est. Daily Usage | Headroom |
|--------|-----------|-----------------|----------|
| Neurons/day | 10,000 | ~100–500 | ~95–99% |

**Neuron usage per model call** (approximate):
- `@cf/meta/llama-3-8b-instruct` — ~1,000–3,000 neurons per conversation turn
- `@cf/baai/bge-small-en-v1.5` (embedding) — ~50–100 neurons per KB entry embed

**First bottleneck**: If Gemini AND Claude are both down simultaneously, cf-chatbot falls to Workers AI. At 10 concurrent fallback conversations, would exhaust the daily budget in ~3 hours. This is an acceptable edge-case risk.

---

### Vectorize (Vector Database)

Used by cf-chatbot for semantic knowledge base search.

| Metric | Free Limit | Est. Monthly Usage | Headroom |
|--------|-----------|-------------------|----------|
| Stored vector dimensions | 5,000,000 | ~15,000–50,000 (50–200 KB entries × 384 dims) | ~99% |
| Queried vector dimensions/month | 30,000,000 | ~200,000–500,000 | ~98–99% |
| Indexes | 100 | 1 (`chatbot-kb-index`) | 99% |

**Index config**: 384 dimensions, cosine similarity, `bge-small-en-v1.5` model.

**First bottleneck**: Vectorize is unlikely to be a bottleneck at any realistic chatbot volume.

---

### Cloudflare Turnstile (Bot Protection)

Used by cf-astro on all public forms (booking, contact, ARCO).

| Metric | Free Limit | Usage | Notes |
|--------|-----------|-------|-------|
| Challenges/verifications | **Unlimited** | — | No limits on free tier |
| Widgets | 20 | 3 (booking, contact, ARCO) | 85% free |

**Turnstile has no practical free tier limit.** It is permanently free at any scale.

---

### Analytics Engine

Used by cf-astro and cf-admin for custom telemetry (page views, booking events, admin actions).

| Metric | Free Limit | Est. Daily Usage | Headroom |
|--------|-----------|-----------------|----------|
| Data points/day | 100,000 | ~500–2,000 | ~98–99% |

**Events written**: page_view, booking_submitted, booking_failed, rate_limit_hit, admin_login, cms_edit, feature_flag_toggle.

---

### Zero Trust (Admin Authentication)

| Metric | Free Limit | Current Usage | Headroom |
|--------|-----------|--------------|----------|
| Users | **50** | ~3 admin users | 94% free |
| Applications | 1 | 1 (`secure.madagascarhotelags.com`) | — |

**First bottleneck**: 50 users. Realistic hotel staff count is well under 10.

---

## External Services

### Supabase (Primary Database)

| Metric | Free Limit | Est. Usage | Headroom |
|--------|-----------|-----------|----------|
| Database size | 500 MB | ~20–50 MB | ~90–96% |
| Active projects | 2 | 1 | 50% |
| Auth MAUs | 50,000 | ~0 (service_role only, no user auth MAUs counted) | ~100% |
| Storage (files) | 1 GB | ~0 (using R2 instead) | ~100% |
| Database egress | 5 GB/month | ~50–200 MB/month | ~96–99% |
| API requests | Unlimited | — | — |

> **Note**: Projects pause after 7 days of inactivity on free plan. Production project should receive regular traffic to prevent pausing.

**First bottleneck**: Database size at 500 MB. Reached when booking history accumulates ~500,000 rows with full PII. At current booking volume, this is 5–10 years away. Upgrade to Supabase Pro ($25/mo) when approaching.

---

### Upstash Redis (Rate Limiting & Caching)

| Metric | Free Limit | Est. Daily Usage | Headroom |
|--------|-----------|-----------------|----------|
| Commands/day | **10,000** | ~500–2,000 | ~80–95% |
| Data size | 256 MB | ~1–5 MB | ~98–99% |
| Concurrent connections | 10 | ~3–4 | ~60–70% |

**Command breakdown**:
- Rate limiting: ~2–4 commands per form submission (SET + GET + EXPIRE)
- Chatbot cache: ~2–3 commands per message (GET + SET)
- Admin rate limits: ~2 commands per API call

**First bottleneck**: 10,000 commands/day — hit at ~2,500 form submissions/day OR ~1,500 chatbot messages/day. At that volume, upgrade to Pay-as-you-go (~$1.50/month for 3x usage).

---

### Resend (Email Delivery)

| Metric | Free Limit | Est. Daily Usage | Headroom |
|--------|-----------|-----------------|----------|
| Emails/day | **100** | ~10–30 | ~70–90% |
| Emails/month | 3,000 | ~300–900 | ~70–90% |
| Custom domains | 1 | 1 (`madagascarhotelags.com`) | Used |
| Webhooks | Unlimited | 1 active | — |

> ⚠️ **Resend is the most likely first bottleneck**. The 100/day limit would be hit at ~50 bookings/day (each booking sends 2 emails: admin + customer). During peak holiday season, this could be reached.

**Upgrade path**: Resend Pro at $20/month provides 50,000 emails/month. This is the only upgrade that might be realistically needed in the first 1–2 years of operation.

**Email types and volume**:
| Email Type | Per Booking/Event | Triggers |
|-----------|------------------|---------|
| `booking_admin` | 1 per booking | New booking submitted |
| `booking_customer` | 1 per booking | New booking submitted |
| `contact` | 1 per contact form | Contact form submitted |
| `arco` | 1 per ARCO request | Privacy rights form submitted |

---

### Sentry (Error Monitoring)

| Metric | Free Limit | Est. Monthly Usage | Headroom |
|--------|-----------|-------------------|----------|
| Errors/month | 5,000 | ~50–200 | ~96–99% |
| Performance transactions | 10,000 | ~500–2,000 | ~80–95% |
| Replays | 50 | 0 (disabled — LCP impact) | — |

**Configuration**: 10% trace sampling (`tracesSampleRate: 0.1`) to conserve free tier. Session Replay explicitly disabled. Error alerts sent to developer email.

**First bottleneck**: 5,000 errors/month — only reached if there are systemic bugs affecting most requests. Normal operation should produce <100 errors/month.

---

### PostHog (Product Analytics)

| Metric | Free Limit | Est. Monthly Usage | Headroom |
|--------|-----------|-------------------|----------|
| Events/month | 1,000,000 | ~5,000–30,000 | ~97–99% |
| Feature flags | Unlimited | — | — |
| Session recordings | 5,000/month | ~0 (disabled for LFPDPPP) | — |

**Events tracked**: page views, booking form starts/completions, language switches, chatbot opens. Session recording disabled due to Mexican privacy law requirements.

---

### GitHub (Source Control)

| Metric | Free Limit | Usage |
|--------|-----------|-------|
| Private repositories | Unlimited | 4 repos (one per service) |
| GitHub Actions minutes | 2,000/month | ~200–500/month |
| Storage | 500 MB (LFS) | <10 MB |

---

## Summary: Free Tier Health at a Glance

| Service | Status | Notes |
|---------|--------|-------|
| Cloudflare Workers | 🟢 Very comfortable | 95–98% headroom |
| Cloudflare D1 | 🟢 Very comfortable | <1% of limits used |
| Cloudflare KV | 🟡 Watch writes | 1,000 writes/day limit; ISR revalidation uses these |
| Cloudflare R2 | 🟢 Very comfortable | 80–95% storage headroom; egress free forever |
| Cloudflare Queues | 🟢 Very comfortable | <1% of limit |
| Workers AI | 🟡 Monitor if chatbot falls back | 10K neurons/day sufficient for normal fallback use |
| Vectorize | 🟢 Very comfortable | <1% of limits |
| Turnstile | 🟢 Unlimited | No practical limit |
| Supabase | 🟢 Very comfortable | 90–96% storage headroom |
| Upstash Redis | 🟡 Watch commands | 10K/day; approaches limit at high form/chat volume |
| **Resend** | 🔴 **Most constrained** | **100/day limit; first real upgrade trigger** |
| Sentry | 🟢 Very comfortable | <4% of error limit |
| PostHog | 🟢 Very comfortable | <3% of event limit |
| GitHub | 🟢 Very comfortable | — |

---

## Upgrade Decision Guide

| Trigger | Service to Upgrade | Cost | When to Act |
|---------|------------------|------|------------|
| >50 bookings/day | Resend Pro | $20/mo | As soon as you're consistently hitting 80+ emails/day |
| >500 admin sessions/day | Workers Standard | $5/mo | Already included if on standard plan |
| >500 MB Supabase DB | Supabase Pro | $25/mo | Monitor in Supabase dashboard |
| >8,000 Upstash commands/day | Upstash Pay-as-you-go | ~$1.50/mo | Automatic |
| >14 days of email message retention | Cloudflare Workers Paid | $5/mo | If DLQ events start expiring |

**Bottom line**: At current and near-term projected hotel volume, the entire stack remains at $0/month. The first realistic paid upgrade is Resend Pro ($20/mo) when booking volume consistently exceeds 50 reservations/day.
