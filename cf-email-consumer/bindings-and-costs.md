# cf-email-consumer — Bindings, Secrets & Costs

> Queue consumer configuration, secrets registry, Resend limits, and cost analysis.

---

## Cloudflare Bindings (wrangler.toml)

```toml
name = "cf-astro-email-consumer"
main = "src/index.ts"
compatibility_date = "2025-03-14"
compatibility_flags = ["nodejs_compat"]

# === QUEUE CONSUMER ===
[[queues.consumers]]
queue = "madagascar-emails"
max_batch_size = 10          # Process up to 10 messages per invocation
max_batch_timeout = 5        # Or wait 5 seconds to accumulate a batch
max_retries = 3              # Retry failed messages up to 3 times

# Dead Letter Queue (PLANNED — not yet configured)
# dead_letter_queue = "madagascar-emails-dlq"
# See: implementations-and-blockers.md for setup steps

# === NO OTHER BINDINGS ===
# cf-email-consumer is intentionally minimal:
# - No D1 (no edge database needed)
# - No KV (no caching needed)
# - No R2 (no file access needed)
# - No AI, Vectorize, Analytics Engine
# Database access is via Supabase postgres.js (external connection)
```

### Queue Consumer Behavior

| Setting | Value | Effect |
|---------|-------|--------|
| `max_batch_size = 10` | 10 messages max per invocation | Allows processing up to 10 emails in one Worker execution |
| `max_batch_timeout = 5` | 5 second wait | Worker waits up to 5s to accumulate a batch before processing |
| `max_retries = 3` | 3 retry attempts | Transient Resend failures get 3 automatic retries |
| Message retention | **24 hours** (free plan) | Unprocessed messages older than 24h are dropped |

### Why max_batch_size = 10?

Resend's rate limit is 100 emails/day on the free tier. Processing 10 emails per invocation ensures:
- Efficient Supabase connection pooling (1 connection per isolate, amortized across 10 messages)
- Reasonable parallelism without overwhelming Resend
- Each message is processed independently — one failure does not block others

---

## Secrets Registry

Set via `wrangler secret put <KEY>` from the `cf-email-consumer/` directory:

| Secret | Purpose | Rotation | Shared With |
|--------|---------|----------|------------|
| `RESEND_API_KEY` | Transactional email API — unique key for consumer | Annually | **Not shared** — consumer has its own Resend API key (different from cf-admin's key) |
| `DATABASE_URL` | Supabase postgres.js connection for audit log updates | On credential rotation | **cf-astro** (same role: `cf_astro_writer`) |
| `SENTRY_DSN` | Error reporting to Sentry | Never | Same DSN as cf-astro/cf-admin |

### Why a Separate Resend API Key?

cf-admin also has a `RESEND_API_KEY` (for security alert emails). These are **different keys** for a reason:
- Separate keys allow per-service usage tracking in the Resend dashboard
- If the consumer key is compromised, it does not expose cf-admin's alert emails
- Independent revocation without affecting other services

### DATABASE_URL: Supavisor Pooler vs Direct Connection

cf-email-consumer uses the **Supavisor pooler** string (not the direct connection):

```
# Pooler (what cf-email-consumer uses):
postgresql://cf_astro_writer.zlvmrepvypucvbyfbpjj:<pw>@aws-0-us-east-1.pooler.supabase.com:6543/postgres

# Direct (what cf-astro uses):
postgresql://cf_astro_writer:<pw>@db.zlvmrepvypucvbyfbpjj.supabase.co:5432/postgres
```

**Why pooler for consumer?** The email consumer processes messages in batches, potentially many concurrent isolates. The pooler handles connection multiplexing. Drizzle is configured with `max: 1` connection per isolate to prevent overwhelming the pool.

---

## Local Development (.dev.vars)

```env
RESEND_API_KEY=re_...
DATABASE_URL=postgresql://cf_astro_writer.zlvmrepvypucvbyfbpjj:password@aws-0-us-east-1.pooler.supabase.com:6543/postgres
SENTRY_DSN=https://...@sentry.io/...
```

### Testing Locally

The consumer can be tested by publishing a message to the queue from cf-astro/cf-admin:

```bash
# Terminal 1: Run the consumer
cd cf-email-consumer
wrangler dev

# Terminal 2: Publish a test message
cd cf-astro
wrangler dev
# Then submit a test booking form → should trigger queue message
```

Alternatively, use the Cloudflare dashboard Queue page to manually send a test message.

---

## Cost & Volume Analysis

### Free Tier: Resend

| Metric | Limit | Est. Usage | Headroom |
|--------|-------|-----------|----------|
| Emails/day | **100** | ~10–30 | ~70–90% |
| Emails/month | 3,000 | ~300–900 | ~70–90% |

**Email volume by type** (approximate at current booking volume):
| Type | Volume | per booking |
|------|--------|------------|
| `booking_admin` | 1 | Every booking |
| `booking_customer` | 1 | Every booking |
| `contact` | ~0.3 | ~3 contact forms per 10 bookings |
| `arco` | ~0.05 | Rare (legal requests) |

**At 20 bookings/day**: ~42–46 emails/day (admin + customer + some contact forms) → 42–46% of free tier.

**Upgrade trigger**: When bookings consistently exceed ~45/day, Resend Pro ($20/month) provides 50,000 emails/month.

### Free Tier: Cloudflare Workers

The consumer is only invoked when queue messages are present:

| Invocation type | Frequency | Workers cost |
|----------------|-----------|-------------|
| Batch of 10 emails | ~2–3 times/day | ~2–3 requests/day |
| Single message invocations | ~5–10 times/day | ~5–10 requests/day |
| **Total** | ~7–13 requests/day | <0.01% of 100K limit |

The consumer has essentially **zero Workers cost** at current volume.

### Free Tier: Cloudflare Queues

| Metric | Limit | Est. Daily Usage | Headroom |
|--------|-------|-----------------|----------|
| Operations/day | 10,000 | ~30–100 | ~99% |

Each email requires 2–3 queue operations (write, read, delete). At 30 emails/day = ~90 operations → 0.9% of free tier.

### Supabase Database (Audit Log)

| Operation | Per Email | Monthly at 900 emails |
|-----------|-----------|----------------------|
| UPDATE email_audit_log (status → 'sent'/'failed') | 1 write | 900 writes |
| SELECT email_audit_log (verify row exists) | 1 read | 900 reads |

Negligible against Supabase free tier limits (5 GB DB storage, unlimited API requests).

---

## Monthly Cost Summary

```
Cloudflare Workers (compute)    $0  (< 1% of free tier)
Cloudflare Queues               $0  (< 1% of free tier)
Resend (email delivery)         $0  (free tier covers current volume)
Supabase (audit log updates)    $0  (shared with cf-astro; negligible writes)
Sentry (error tracking)         $0  (shared DSN; consumer errors rare)
─────────────────────────────────────────────────
cf-email-consumer TOTAL         $0/month

First upgrade trigger:
  >45 bookings/day → Resend Pro at $20/month
```

---

## Dead Letter Queue (DLQ) — Current Status

The DLQ (`madagascar-emails-dlq`) is **planned but not yet active**. When a message exhausts all 3 retries, it is currently dropped silently.

**Impact**: If Resend is down for more than 3 retry cycles AND messages expire from the 24h queue retention, booking confirmation emails are permanently lost.

**Remediation** (see [Implementations & Blockers](../implementations-and-blockers.md)):
1. Create `madagascar-emails-dlq` queue in Cloudflare dashboard
2. Add `dead_letter_queue = "madagascar-emails-dlq"` to consumer config
3. Create a DLQ consumer function that sends a Sentry alert for every DLQ message
4. Redeploy with updated wrangler.toml

This is the **highest priority operational risk** in the current email architecture.
