# cf-admin — Bindings & Secrets

All Cloudflare bindings, public vars, secrets, local dev setup, and cross-service sync requirements.

---

## Cloudflare Bindings (`wrangler.toml`)

### Annotated `wrangler.toml` Bindings Section

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
database_id = "7fca2a07-d7b4-449d-b446-408f9187d3ca"

# === KV NAMESPACES ===
# Session storage — isolated from cf-astro (different namespace)
[[kv_namespaces]]
binding = "SESSION"
id = "<cf-admin-session-kv-id>"

# === R2 OBJECT STORAGE ===
# Image bucket — shared with cf-astro (same bucket name)
[[r2_buckets]]
binding = "IMAGES"
bucket_name = "madagascar-images"

# === CLOUDFLARE QUEUES ===
# Email queue producer — cf-email-consumer is the consumer
[[queues.producers]]
binding = "EMAIL_QUEUE"
queue = "madagascar-emails"

# === ANALYTICS ENGINE ===
[[analytics_engine_datasets]]
binding = "ANALYTICS"
dataset = "madagascar_analytics"

# === SERVICE BINDINGS ===
[[services]]
binding = "CHATBOT_SERVICE"
service = "cf-chatbot"           # Zero-latency internal call to cf-chatbot worker

[[services]]
binding = "ASTRO_SERVICE"
service = "cf-astro"             # Zero-latency internal call for ISR revalidation

# === CRON TRIGGERS ===
[triggers]
crons = ["*/5 * * * *"]          # Every 5 minutes — 288 invocations/day

# === OBSERVABILITY ===
[observability]
enabled = true
```

### Binding Reference Table

| Binding | Type | Resource | Purpose |
|---------|------|----------|---------|
| `DB` | D1 Database | `madagascar-db` (`7fca2a07-d7b4-449d-b446-408f9187d3ca`) | Audit logs, PLAC config, CMS content, login forensics, booking shadow state, feature flags, diagnostic runs |
| `IMAGES` | R2 Bucket | `madagascar-images` | Gallery and hero image storage; CDN served via `cdn.madagascarhotelags.com` |
| `SESSION` | KV Namespace | cf-admin session namespace | KV sessions (`session:{id}`), reverse index (`user-session:{uid}:{sid}`), revocation flags (`revoked:{uid}`), cron timestamp (`cf-audit-last-synced`) |
| `EMAIL_QUEUE` | Queue Producer | `madagascar-emails` | Async email dispatch via Resend for security alerts |
| `ANALYTICS` | Analytics Engine | `madagascar_analytics` | Optional admin usage metrics |
| `CHATBOT_SERVICE` | Service Binding | cf-chatbot worker | Zero-latency proxy to chatbot admin API; no HTTP call overhead |
| `ASTRO_SERVICE` | Service Binding | cf-astro worker | Zero-latency ISR revalidation calls after CMS edits |

---

## Public Vars (`wrangler.toml [vars]`)

These are non-secret values committed to `wrangler.toml`. Do not move secrets here.

```toml
[vars]
SITE_URL                     = "https://secure.madagascarhotelags.com"
PUBLIC_SUPABASE_URL          = "https://zlvmrepvypucvbyfbpjj.supabase.co"
PUBLIC_ASTRO_URL             = "https://madagascarhotelags.com"
PUBLIC_CDN_URL               = "https://cdn.madagascarhotelags.com"
PUBLIC_SENTRY_DSN            = "https://389bb...@o4510752...ingest.us.sentry.io/4511193..."
ADMIN_EMAIL                  = "mascotasmadagascar@gmail.com"
SENDER_EMAIL                 = "team@madagascarhotelags.com"
SESSION_REFRESH_INTERVAL_MS  = "1800000"    # 30 minutes — role re-check interval
SESSION_MAX_LIFETIME_MS      = "86400000"   # 24 hours — hard session expiry
CF_ACCOUNT_ID                = "320d1ebab5143958d2acd481ea465f52"
CF_D1_DATABASE_ID            = "7fca2a07-d7b4-449d-b446-408f9187d3ca"
CF_R2_BUCKET_NAME            = "madagascar-images"
CF_QUEUE_NAME                = "madagascar-emails"
CF_TEAM_NAME                 = "mascotas"
CF_ACCESS_AUD                = "680d415033ba49284452da6f51bc8b7ee9aa23345ac95c53cf49761140b13088"
LOCAL_DEV_ADMIN_EMAIL        = "harshil.8136@gmail.com"
```

**Notes:**
- `CF_ACCESS_AUD` is the audience tag for the CF Zero Trust application protecting `secure.madagascarhotelags.com`. It is not a secret.
- `LOCAL_DEV_ADMIN_EMAIL` is only functional when `SITE_URL` contains `localhost`. It is non-sensitive.
- `SESSION_REFRESH_INTERVAL_MS` and `SESSION_MAX_LIFETIME_MS` control session timing — changing these requires a full redeployment.

---

## Secrets Registry

All secrets are set via `wrangler secret put <KEY>`. In local dev, they are stored in `.dev.vars` (gitignored).

### Core Service Secrets

| Secret | Purpose | Rotation Policy |
|--------|---------|-----------------|
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase service_role JWT — bypasses RLS for full admin database access | Annually or on compromise |
| `REVALIDATION_SECRET` | Authorization token for ISR revalidation calls to cf-astro | Annually; **must match cf-astro** |
| `IP_HASH_SECRET` | HMAC secret for privacy-safe IP hashing in audit logs | Do not rotate — breaks historical correlation |
| `RESEND_API_KEY` | Security alert email dispatch via Resend | Annually; separate key from cf-email-consumer |
| `SENTRY_AUTH_TOKEN` | Source map upload during CI build (not needed at runtime) | Annually |
| `SENTRY_ORG_SLUG` | Sentry organization slug | Change only if org changes |
| `SENTRY_PROJECT_SLUG` | Sentry project slug | Change only if project changes |

### Upstash Redis Secrets

| Secret | Purpose |
|--------|---------|
| `UPSTASH_REDIS_REST_URL` | Upstash Redis REST endpoint (shared with other services — different key prefixes) |
| `UPSTASH_REDIS_REST_TOKEN` | Authentication token for Upstash Redis |

### Cloudflare API Secrets

| Secret | Purpose | Required Scope |
|--------|---------|---------------|
| `CLOUDFLARE_API_TOKEN` | CF API access for diagnostics | Varies |
| `CLOUDFLARE_ZONE_ID` | `c73b1ccd7f03999ea419ef8177fa68d4` — zone for cache purge | Zone-specific |
| `CF_API_TOKEN_READ_LOGS` | Read CF Zero Trust audit logs (used by cron) | `Access: Audit Logs: Read` (read-only) |
| `CF_API_TOKEN_ZT_WRITE` | Revoke CF Access sessions (Layer 3 force-logout) | `Access: Revoke Tokens` (write) |

### Chatbot Integration Secrets

| Secret | Purpose | Must Match |
|--------|---------|-----------|
| `CHATBOT_ADMIN_API_KEY` | `X-Admin-Key` header for cf-chatbot admin API | `ADMIN_API_KEY` in cf-chatbot |
| `CHATBOT_WORKER_URL` | `https://charlar.madagascarhotelags.com` — HTTP fallback if CHATBOT_SERVICE unavailable | — |

---

## Secrets That Require Cross-Service Coordination

When rotating these secrets, **both services must be updated simultaneously** to avoid service disruption:

| Secret | Used In cf-admin As | Used In Other Service As | Consequence of Drift |
|--------|--------------------|--------------------------|--------------------|
| `REVALIDATION_SECRET` | Authorization header on ASTRO_SERVICE calls | `REVALIDATION_SECRET` in cf-astro | ISR revalidation fails — stale content shown to visitors |
| `CHATBOT_ADMIN_API_KEY` | `X-Admin-Key` header on CHATBOT_SERVICE calls | `ADMIN_API_KEY` in cf-chatbot | All chatbot admin operations return 401 |

**Rotation procedure for coordinated secrets:**
1. Generate new secret value
2. Update cf-admin: `wrangler secret put <SECRET_NAME>`
3. Update the other service: `wrangler secret put <SECRET_NAME>` (in that service's directory)
4. Deploy both services simultaneously (or within seconds of each other)

---

## Local Development (`.dev.vars`)

Copy the template below to `cf-admin/.dev.vars` (this file is gitignored):

```env
# Supabase
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Shared secrets (must match cf-astro .dev.vars)
REVALIDATION_SECRET=your-local-revalidation-secret

# Security
IP_HASH_SECRET=your-random-32-char-salt

# Upstash Redis (shared — use your own account in dev or a shared dev instance)
UPSTASH_REDIS_REST_URL=https://us1-example.upstash.io
UPSTASH_REDIS_REST_TOKEN=Abcd...

# Cloudflare API (can use personal API token with limited scope for local testing)
CLOUDFLARE_API_TOKEN=your-cf-api-token
CLOUDFLARE_ZONE_ID=c73b1ccd7f03999ea419ef8177fa68d4
CF_API_TOKEN_READ_LOGS=your-read-only-cf-api-token
CF_API_TOKEN_ZT_WRITE=your-zt-write-cf-api-token

# Email (security alerts — use a test API key in local dev)
RESEND_API_KEY=re_test_...

# Sentry (optional in local dev)
SENTRY_AUTH_TOKEN=sntrys_...
SENTRY_ORG_SLUG=your-org
SENTRY_PROJECT_SLUG=cf-admin

# Chatbot integration
CHATBOT_ADMIN_API_KEY=your-admin-api-key
CHATBOT_WORKER_URL=https://charlar.madagascarhotelags.com
```

**Important:** Do not set `SITE_URL` or `PUBLIC_ASTRO_URL` in `.dev.vars`. These are `[vars]` in `wrangler.toml` and will be overridden by Wrangler correctly. Setting `PUBLIC_ASTRO_URL` locally could accidentally trigger revalidation calls against the production cf-astro site.

---

## Setting Production Secrets

```bash
# From the cf-admin project directory
wrangler secret put SUPABASE_SERVICE_ROLE_KEY
wrangler secret put REVALIDATION_SECRET
wrangler secret put IP_HASH_SECRET
wrangler secret put UPSTASH_REDIS_REST_URL
wrangler secret put UPSTASH_REDIS_REST_TOKEN
wrangler secret put CLOUDFLARE_API_TOKEN
wrangler secret put CLOUDFLARE_ZONE_ID
wrangler secret put CF_API_TOKEN_READ_LOGS
wrangler secret put CF_API_TOKEN_ZT_WRITE
wrangler secret put RESEND_API_KEY
wrangler secret put SENTRY_AUTH_TOKEN
wrangler secret put SENTRY_ORG_SLUG
wrangler secret put SENTRY_PROJECT_SLUG
wrangler secret put CHATBOT_ADMIN_API_KEY
wrangler secret put CHATBOT_WORKER_URL

# Verify all secrets are set
wrangler secret list
```

---

## Binding IDs Quick Reference

| Binding | ID / Name | Shared With |
|---------|----------|-------------|
| D1 `DB` | `7fca2a07-d7b4-449d-b446-408f9187d3ca` (`madagascar-db`) | cf-astro (same database) |
| R2 `IMAGES` | `madagascar-images` | cf-astro (same bucket) |
| Queue `EMAIL_QUEUE` | `madagascar-emails` | cf-email-consumer (consumer side) |
| Analytics `ANALYTICS` | `madagascar_analytics` | cf-astro (same dataset) |
| KV `SESSION` | cf-admin exclusive namespace | Not shared |
| Service `CHATBOT_SERVICE` | cf-chatbot worker | — |
| Service `ASTRO_SERVICE` | cf-astro worker | — |
