# cf-astro — Bindings & Secrets

Complete documentation of all Cloudflare bindings, public environment variables, and secrets required by the cf-astro Worker.

---

## Cloudflare Bindings (wrangler.toml)

### D1 Databases

```toml
[[d1_databases]]
binding = "DB"
database_name = "madagascar-db"
database_id = "7fca2a07-d7b4-449d-b446-408f9187d3ca"
```

| Binding | Database name | ID | Purpose |
|---------|--------------|-----|---------|
| `DB` | `madagascar-db` | `7fca2a07-d7b4-449d-b446-408f9187d3ca` | Feature flags (`admin_feature_flags`), CMS content blocks (`cms_content`), ARCO legal records (`legalRequests`) |

**Accessed by**: Middleware (feature flags), `/api/revalidate` (CMS writes), `/api/arco/submit` (legal records), `/api/arco/get-document` (ticket lookup), `/api/admin/generate-blog-draft`, `/api/admin/generate-faqs`

---

### KV Namespaces

```toml
[[kv_namespaces]]
binding = "ISR_CACHE"
id = "d9cea8c7e20f4b328b8cb3b04104138c"

[[kv_namespaces]]
binding = "SESSION"
id = "bee123e795504473accf58ac5b6de13d"
```

| Binding | Namespace name | ID | Purpose |
|---------|---------------|-----|---------|
| `ISR_CACHE` | `cf-astro-isr-cache` | `d9cea8c7e20f4b328b8cb3b04104138c` | Cached page HTML. Keys: `isr:{path}#{buildId}`. TTL: 24 hours. |
| `SESSION` | `cf-astro-session` | `bee123e795504473accf58ac5b6de13d` | Astro 6 adapter session storage |

**ISR_CACHE accessed by**: Middleware (read + write), `/api/revalidate` (delete), `/api/health` (latency check)

---

### R2 Buckets

```toml
[[r2_buckets]]
binding = "IMAGES"
bucket_name = "madagascar-images"

[[r2_buckets]]
binding = "ARCO_DOCS"
bucket_name = "arco-documents"
```

| Binding | Bucket name | Access | Purpose |
|---------|------------|--------|---------|
| `IMAGES` | `madagascar-images` | Private (served via Worker) | CMS images and gallery photos — shared with cf-admin |
| `ARCO_DOCS` | `arco-documents` | Private (admin-only retrieval) | Identity documents submitted with ARCO legal requests |

**IMAGES accessed by**: `/api/media/[...path]` (dev-only proxy)
**ARCO_DOCS accessed by**: `/api/arco/submit` (upload), `/api/arco/get-document` (admin download)

---

### Queue Producers

```toml
[[queues.producers]]
binding = "EMAIL_QUEUE"
queue = "madagascar-emails"
```

| Binding | Queue name | Purpose |
|---------|-----------|---------|
| `EMAIL_QUEUE` | `madagascar-emails` | Async email dispatch — consumed by `cf-email-consumer` Worker |

**Accessed by**: `/api/booking` (admin + customer emails), `/api/contact` (contact form email), `/api/privacy/arco` (ARCO notification)

**Consumer**: `cf-email-consumer` Worker processes the queue and sends emails via Resend API.

---

### Analytics Engine

```toml
[[analytics_engine_datasets]]
binding = "ANALYTICS"
dataset = "madagascar_analytics"
```

| Binding | Dataset name | Purpose |
|---------|-------------|---------|
| `ANALYTICS` | `madagascar_analytics` | Edge telemetry: page views, booking events, CTA tracking |

**Accessed by**: Middleware (page_view), `/api/booking` (booking_success/failed), `/api/analytics/track` (CTA events), `/api/test-services` (smoke test)

---

### Workers AI

```toml
[ai]
binding = "AI"
```

| Binding | Model | Purpose |
|---------|-------|---------|
| `AI` | `@cf/meta/llama-3-8b-instruct` | Blog post generation (800 words) and FAQ generation (12 Q&As) |

**Accessed by**: `/api/admin/generate-blog-draft`, `/api/admin/generate-faqs`

**Free quota**: 10,000 neurons per day. Blog/FAQ generation is triggered manually by admin and used rarely.

---

## Public Variables (wrangler.toml `[vars]`)

These are non-secret configuration values baked into the Worker at deploy time. Safe to commit.

```toml
[vars]
SITE_URL = "https://madagascarhotelags.com"
POSTHOG_HOST = "https://us.i.posthog.com"
DEFAULT_LOCALE = "es"
PUBLIC_SUPABASE_URL = "https://zlvmrepvypucvbyfbpjj.supabase.co"
ADMIN_EMAIL = "booking@madagascarhotelags.com"
SENDER_EMAIL = "Hotel Madagascar <booking@madagascarhotelags.com>"
PUBLIC_SUPABASE_ANON_KEY = "sb_publishable_JC5dlnv64ReWypNQ7M_vyQ_SAMOe-Dh"
UPSTASH_REDIS_REST_URL = "https://modest-mastiff-88856.upstash.io"
PUBLIC_TURNSTILE_SITE_KEY = "0x4AAAAAAC00HXfps2jjR_Bj"
```

| Variable | Value | Notes |
|----------|-------|-------|
| `SITE_URL` | `https://madagascarhotelags.com` | Canonical origin |
| `POSTHOG_HOST` | `https://us.i.posthog.com` | PostHog proxy target |
| `DEFAULT_LOCALE` | `es` | Primary language |
| `PUBLIC_SUPABASE_URL` | `https://zlvmrepvypucvbyfbpjj.supabase.co` | Supabase project URL |
| `ADMIN_EMAIL` | `booking@madagascarhotelags.com` | Admin notification recipient |
| `SENDER_EMAIL` | `Hotel Madagascar <booking@madagascarhotelags.com>` | From address for outgoing emails |
| `PUBLIC_SUPABASE_ANON_KEY` | `sb_publishable_JC5dlnv64ReWypNQ7M_vyQ_SAMOe-Dh` | Safe to commit — RLS blocks anon reads on PII tables |
| `UPSTASH_REDIS_REST_URL` | `https://modest-mastiff-88856.upstash.io` | Rate limiting Redis instance |
| `PUBLIC_TURNSTILE_SITE_KEY` | `0x4AAAAAAC00HXfps2jjR_Bj` | Cloudflare Turnstile site key (client-side widget) |

---

## Secrets (`wrangler secret put`)

Secrets are stored in Cloudflare's encrypted secrets store and injected into the Worker env at runtime. Never committed to source control.

```bash
# Add a secret
npx wrangler secret put SECRET_NAME

# List secrets (names only, values are never shown)
npx wrangler secret list

# Delete a secret
npx wrangler secret delete SECRET_NAME
```

| Secret | Purpose | Shared with |
|--------|---------|-------------|
| `DATABASE_URL` | Supabase PostgreSQL direct connection string using `cf_astro_writer` role (least-privilege) | `cf-email-consumer` (same role) |
| `DATABASE_URL_ADMIN` | Postgres superuser connection string. Emergency use only — not in runtime hot path. | — |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service role JWT for SDK usage if needed | — |
| `RESEND_API_KEY` | Transactional email API key for Resend | — |
| `RESEND_WEBHOOK_SECRET` | Svix HMAC-SHA256 signing secret for Resend delivery webhooks | — |
| `REVALIDATION_SECRET` | Bearer token for ISR cache purge and analytics summary auth | `cf-admin` (must match exactly) |
| `ADMIN_AI_SECRET` | Bearer token for blog/FAQ AI generation endpoints | — |
| `HEALTH_CHECK_SECRET` | Bearer token for health probe and Analytics Engine smoke test | — |
| `ARCO_ADMIN_SECRET` | Header secret for ARCO document retrieval endpoint (admin only) | — |
| `TURNSTILE_SECRET_KEY` | Cloudflare Turnstile server-side secret for CAPTCHA verification | — |
| `UPSTASH_REDIS_REST_TOKEN` | Upstash Redis REST API auth token for rate limiting | `cf-admin`, `cf-chatbot` (different key prefixes) |
| `BETTERSTACK_SOURCE_TOKEN` | BetterStack Logtail source token for structured logging | — |
| `SENTRY_AUTH_TOKEN` | Sentry auth token for source map upload at build time (CI/CD only — not in Worker runtime) | — |

---

## Local Development Setup

### `.dev.vars` File

Create `cf-astro/.dev.vars` (never commit — gitignored):

```ini
# Database
DATABASE_URL=postgresql://cf_astro_writer:your-password@db.zlvmrepvypucvbyfbpjj.supabase.co:5432/postgres?sslmode=require
DATABASE_URL_ADMIN=postgresql://postgres:your-admin-password@db.zlvmrepvypucvbyfbpjj.supabase.co:5432/postgres?sslmode=require

# Cloudflare Turnstile (always-pass test key for local dev)
TURNSTILE_SECRET_KEY=1x0000000000000000000000000000000AA

# Auth secrets (any value works locally)
REVALIDATION_SECRET=dev-revalidation-secret
HEALTH_CHECK_SECRET=dev-health-secret
ADMIN_AI_SECRET=dev-ai-secret
ARCO_ADMIN_SECRET=dev-arco-secret

# Upstash Redis
UPSTASH_REDIS_REST_TOKEN=your-upstash-token

# Email
RESEND_API_KEY=re_your_resend_api_key
RESEND_WEBHOOK_SECRET=your-resend-webhook-secret

# Monitoring
BETTERSTACK_SOURCE_TOKEN=your-betterstack-token
# SENTRY_AUTH_TOKEN is CI-only, not needed locally
```

**Note**: Wrangler reads `.dev.vars` automatically when running `wrangler dev`. Do not prefix variable names with `PUBLIC_` in `.dev.vars` — that prefix is only for Astro's client-side exposure in `astro.config.ts`.

---

## Cross-Project Secret Sharing

Secrets shared between projects must be identical — mismatches cause auth failures.

| Secret | Projects that share it | Sync requirement |
|--------|----------------------|-----------------|
| `REVALIDATION_SECRET` | cf-astro, cf-admin | Must match exactly in both Workers |
| `UPSTASH_REDIS_REST_TOKEN` | cf-astro, cf-admin, cf-chatbot | Same token, different key prefixes per project |
| `DATABASE_URL` (`cf_astro_writer` role) | cf-astro, cf-email-consumer | Same connection string, same role |

---

## Secret Rotation Notes

| Secret | Rotation trigger | Steps |
|--------|-----------------|-------|
| `DATABASE_URL` | Password rotation or role re-creation | 1. Update Supabase password, 2. `wrangler secret put DATABASE_URL` in cf-astro and cf-email-consumer, 3. Redeploy both |
| `REVALIDATION_SECRET` | On demand or security incident | 1. Generate new secret, 2. `wrangler secret put` in both cf-astro and cf-admin simultaneously, 3. Redeploy both |
| `RESEND_API_KEY` | Resend key compromise | 1. Revoke in Resend dashboard, 2. Create new key, 3. `wrangler secret put RESEND_API_KEY` |
| `UPSTASH_REDIS_REST_TOKEN` | Redis token compromise | 1. Rotate in Upstash dashboard, 2. Update in all three projects (cf-astro, cf-admin, cf-chatbot) |
| `ARCO_ADMIN_SECRET` | On demand | 1. Generate new secret, 2. `wrangler secret put ARCO_ADMIN_SECRET`, 3. Update in any admin tooling that uses the endpoint |
| `SENTRY_AUTH_TOKEN` | CI/CD only; never in runtime | Rotate in Sentry → update CI environment variable (not wrangler secret) |

---

## Wrangler Configuration Skeleton

```toml
name = "cf-astro"
compatibility_date = "2025-04-01"
compatibility_flags = ["nodejs_compat"]
main = "./dist/_worker.js"
assets = { directory = "./dist", binding = "ASSETS" }

[vars]
SITE_URL = "https://madagascarhotelags.com"
# ... (see Public Variables section above)

[[d1_databases]]
binding = "DB"
database_name = "madagascar-db"
database_id = "7fca2a07-d7b4-449d-b446-408f9187d3ca"

[[kv_namespaces]]
binding = "ISR_CACHE"
id = "d9cea8c7e20f4b328b8cb3b04104138c"

[[kv_namespaces]]
binding = "SESSION"
id = "bee123e795504473accf58ac5b6de13d"

[[r2_buckets]]
binding = "IMAGES"
bucket_name = "madagascar-images"

[[r2_buckets]]
binding = "ARCO_DOCS"
bucket_name = "arco-documents"

[[queues.producers]]
binding = "EMAIL_QUEUE"
queue = "madagascar-emails"

[[analytics_engine_datasets]]
binding = "ANALYTICS"
dataset = "madagascar_analytics"

[ai]
binding = "AI"
```

Secrets are added separately via `wrangler secret put` and do not appear in `wrangler.toml`.
