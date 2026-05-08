# cf-admin — Bindings & Secrets

> Complete wrangler.toml documentation, all secrets with purpose and rotation, and environment variable reference.

---

## Cloudflare Bindings (wrangler.toml)

### Annotated wrangler.toml

```toml
name = "cf-admin-madagascar"
main = "dist/server/index.js"       # Astro SSR output
compatibility_date = "2025-03-14"
compatibility_flags = ["nodejs_compat"]

# === D1 DATABASE ===
# Shared with cf-astro — same physical database, different table prefixes
[[d1_databases]]
binding = "DB"
database_name = "madagascar-db"
database_id = "7fca2a07-d7b4-449d-b446-408f9187d3ca"    # ← Must match cf-astro exactly

# === KV NAMESPACES ===
# Session storage — ISOLATED from cf-astro (different namespace ID)
[[kv_namespaces]]
binding = "SESSION"
id = "ba82eecc6f5a4956ad63178b203a268f"    # cf-admin-session (NOT the same as cf-astro)

# === R2 OBJECT STORAGE ===
# Image bucket — shared with cf-astro (same bucket name)
[[r2_buckets]]
binding = "IMAGES"
bucket_name = "madagascar-images"           # ← Must match cf-astro bucket name

# === CLOUDFLARE QUEUES ===
# Email queue producer — cf-email-consumer is the consumer
[[queues.producers]]
binding = "EMAIL_QUEUE"
queue = "madagascar-emails"

# === ANALYTICS ENGINE ===
# Custom telemetry — shared dataset with cf-astro
[[analytics_engine_datasets]]
binding = "ANALYTICS"
dataset = "madagascar_analytics"           # ← Must match cf-astro dataset name

# === CRON TRIGGERS ===
# CF Access audit log polling (pulls ZT auth events into D1)
[triggers]
crons = ["*/5 * * * *"]

# === OBSERVABILITY ===
[observability]
enabled = true                            # Workers Logpush + trace sampling

# === ENVIRONMENT VARIABLES (non-secret) ===
[vars]
SITE_URL = "https://secure.madagascarhotelags.com"
PUBLIC_SUPABASE_URL = "https://zlvmrepvypucvbyfbpjj.supabase.co"
PUBLIC_ASTRO_URL = "https://madagascarhotelags.com"
PUBLIC_CDN_URL = "https://cdn.madagascarhotelags.com"
CF_TEAM_NAME = "mascotas"
CF_ACCESS_AUD = "<audience-tag-from-cf-dashboard>"
CF_ACCOUNT_ID = "320d1ebab5143958d2acd481ea465f52"
CF_D1_DATABASE_ID = "7fca2a07-d7b4-449d-b446-408f9187d3ca"
CF_R2_BUCKET_NAME = "madagascar-images"
CF_QUEUE_NAME = "madagascar-emails"
LOCAL_DEV_ADMIN_EMAIL = "harshil.8136@gmail.com"
CHATBOT_WORKER_URL = "https://charlar.madagascarhotelags.com"
```

---

## Secrets Registry

All secrets are stored encrypted via `wrangler secret put <KEY>`. In local dev, they go in `.dev.vars` (gitignored).

### Core Service Secrets

| Secret | Purpose | Rotation | Shared With |
|--------|---------|----------|------------|
| `SUPABASE_SERVICE_ROLE_KEY` | Full Supabase access (bypasses RLS) | Annually or on compromise | cf-chatbot |
| `REVALIDATION_SECRET` | Authorization token for cf-astro ISR webhook | Annually | **cf-astro** (must match) |
| `IP_HASH_SECRET` | Salt for SHA-256 IP hashing in audit logs | Never (rotation breaks historical correlation) | — |
| `RESEND_API_KEY` | Email dispatch for security alerts | Annually | cf-email-consumer (separate key) |
| `SENTRY_AUTH_TOKEN` | Source map upload (build time only) | Annually | — |
| `SENTRY_DSN` | Error reporting endpoint | Never (DSN is not a secret, but stored here) | — |

### Upstash Redis Secrets

| Secret | Purpose |
|--------|---------|
| `UPSTASH_REDIS_REST_URL` | Upstash Redis REST endpoint |
| `UPSTASH_REDIS_REST_TOKEN` | Authentication token for Upstash |

### Cloudflare API Secrets

| Secret | Purpose | Scope |
|--------|---------|-------|
| `CF_API_TOKEN_READ_LOGS` | Read Zero Trust audit logs (cron) | Read-only: `Access: Audit Logs: Read` |
| `CF_API_TOKEN_ZT_WRITE` | Revoke CF Access sessions (force-kick Layer 3) | Write: `Access: Revoke Tokens` |

### Chatbot Integration Secrets

| Secret | Purpose | Must Match |
|--------|---------|-----------|
| `CHATBOT_ADMIN_API_KEY` | X-Admin-Key header for cf-chatbot admin API | `ADMIN_API_KEY` in cf-chatbot |

---

## Local Development (.dev.vars)

Copy this template to `cf-admin/.dev.vars` (gitignored):

```env
# Supabase
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Shared secrets (must match cf-astro)
REVALIDATION_SECRET=your-revalidation-secret

# Security
IP_HASH_SECRET=your-random-32-char-salt

# Upstash
UPSTASH_REDIS_REST_URL=https://us1-example.upstash.io
UPSTASH_REDIS_REST_TOKEN=Abcd...

# Cloudflare API
CF_API_TOKEN_READ_LOGS=your-read-only-cf-api-token
CF_API_TOKEN_ZT_WRITE=your-zt-write-cf-api-token

# Email (for security alerts only)
RESEND_API_KEY=re_...

# Sentry
SENTRY_DSN=https://...@sentry.io/...
SENTRY_AUTH_TOKEN=sntrys_...

# Chatbot integration
CHATBOT_ADMIN_API_KEY=your-admin-api-key

# Optional: Security alert destination
SECURITY_ALERT_EMAIL=harshil.8136@gmail.com
```

> **Critical**: `SITE_URL` and `PUBLIC_ASTRO_URL` are `[vars]` in wrangler.toml — do NOT set them in `.dev.vars`. Setting `PUBLIC_ASTRO_URL` locally would cause CMS revalidation calls to hit the production cf-astro instance during local development.

---

## Deployment: Setting Production Secrets

```bash
# From the cf-admin directory
wrangler secret put SUPABASE_SERVICE_ROLE_KEY
wrangler secret put REVALIDATION_SECRET
wrangler secret put IP_HASH_SECRET
wrangler secret put UPSTASH_REDIS_REST_URL
wrangler secret put UPSTASH_REDIS_REST_TOKEN
wrangler secret put CF_API_TOKEN_READ_LOGS
wrangler secret put CF_API_TOKEN_ZT_WRITE
wrangler secret put RESEND_API_KEY
wrangler secret put SENTRY_AUTH_TOKEN
wrangler secret put SENTRY_DSN
wrangler secret put CHATBOT_ADMIN_API_KEY
wrangler secret put SECURITY_ALERT_EMAIL
```

**Verify** all secrets are set:
```bash
wrangler secret list
```

---

## Binding IDs Quick Reference

| Binding | ID / Name | Shared? |
|---------|----------|---------|
| D1 `DB` | `7fca2a07-d7b4-449d-b446-408f9187d3ca` | Yes — same as cf-astro |
| KV `SESSION` | `ba82eecc6f5a4956ad63178b203a268f` | No — cf-admin exclusive |
| R2 `IMAGES` | `madagascar-images` (bucket name) | Yes — same as cf-astro |
| Queue `EMAIL_QUEUE` | `madagascar-emails` | Yes — produces to same queue as cf-astro |
| Analytics `ANALYTICS` | `madagascar_analytics` (dataset) | Yes — same dataset as cf-astro |

---

## Secrets That Require Coordination

When rotating these secrets, **both services must be updated simultaneously**:

| Secret | Primary Service | Also In | Consequence of Drift |
|--------|---------------|---------|---------------------|
| `REVALIDATION_SECRET` | cf-admin | cf-astro | ISR revalidation fails silently → stale content |
| `CHATBOT_ADMIN_API_KEY` | cf-admin | cf-chatbot (as `ADMIN_API_KEY`) | All chatbot admin operations return 401 |
