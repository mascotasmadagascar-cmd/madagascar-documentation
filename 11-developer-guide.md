# 11 — Developer Guide

> Setup, secrets management, deployment workflow, and local development.

---

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Node.js | ≥ 18.x (LTS) | Runtime |
| npm | ≥ 9.x | Package manager |
| Wrangler | ≥ 4.x | Cloudflare CLI |
| Git | ≥ 2.x | Version control |

### Install Wrangler globally
```bash
npm install -g wrangler
wrangler login  # Authenticate with Cloudflare account
```

---

## Repository Structure

```
Madagascar Project/
├── cf-astro/           # Public website (Astro 6 + Preact)
├── cf-admin/           # Admin dashboard (Astro 6 + Preact)
├── cf-chatbot/         # AI chatbot (Vanilla TS)
├── cf-email-consumer/  # Queue consumer (Vanilla TS)
├── documentation/      # This documentation hub
└── .agents/            # AI agent skills & config
```

Each service is an independent project with its own `package.json`, `wrangler.toml`/`wrangler.jsonc`, and deployment pipeline.

---

## Local Development Setup

### 1. cf-astro (Public Website)

```bash
cd cf-astro
npm install

# Create .dev.vars with local secrets
cat > .dev.vars << 'EOF'
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_KEY=eyJ...
DATABASE_URL=postgresql://...
BETTERSTACK_SOURCE_TOKEN=...
PUBLIC_SENTRY_DSN=https://xxx@xxx.ingest.us.sentry.io/xxx
PUBLIC_TURNSTILE_SITE_KEY=...
TURNSTILE_SECRET_KEY=...
UPSTASH_REDIS_REST_URL=https://...
UPSTASH_REDIS_REST_TOKEN=...
REVALIDATION_SECRET=...
EOF

# Start dev server
npm run dev   # → http://localhost:4321
```

### 2. cf-admin (Admin Dashboard)

```bash
cd cf-admin
npm install

# Create .dev.vars
cat > .dev.vars << 'EOF'
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_KEY=eyJ...
SITE_URL=http://localhost:4322
LOCAL_DEV_ADMIN_EMAIL=dev@example.com
CF_ACCESS_AUD=xxx
CF_TEAM_NAME=xxx
SESSION_MAX_LIFETIME_MS=86400000
RESEND_API_KEY=re_xxx
SECURITY_ALERT_EMAIL=admin@example.com
CHATBOT_WORKER_URL=http://localhost:8787
CHATBOT_ADMIN_KEY=xxx
REVALIDATION_SECRET=xxx
UPSTASH_REDIS_REST_URL=https://...
UPSTASH_REDIS_REST_TOKEN=...
EOF

# Start dev server
npm run dev   # → http://localhost:4322
```

**Local Dev Note**: CF Access is unavailable locally. The middleware detects `localhost` in `SITE_URL` and requires explicit login via `/api/auth/dev-login` instead of auto-bootstrapping from CF headers.

### 3. cf-chatbot (AI Chatbot)

```bash
cd cf-chatbot
npm install

# Create .dev.vars
cat > .dev.vars << 'EOF'
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_KEY=eyJ...
GEMINI_API_KEY=AIza...
ANTHROPIC_API_KEY=sk-ant-...
ADMIN_API_KEY=xxx
WHATSAPP_VERIFY_TOKEN=xxx
WHATSAPP_API_TOKEN=xxx
WHATSAPP_PHONE_NUMBER_ID=xxx
ALLOWED_ORIGINS=http://localhost:4321,http://localhost:4322
UPSTASH_REDIS_REST_URL=https://...
UPSTASH_REDIS_REST_TOKEN=...
EOF

# Start dev server (Wrangler local mode)
npm run dev   # → http://localhost:8787
```

### 4. cf-email-consumer (Queue Consumer)

```bash
cd cf-email-consumer
npm install

# Create .dev.vars
cat > .dev.vars << 'EOF'
DATABASE_URL=postgresql://...
RESEND_API_KEY=re_xxx
SENTRY_DSN=https://xxx@xxx.ingest.us.sentry.io/xxx
EOF

# Start dev server
npm run dev   # → http://localhost:8788
```

**Note**: Queue consumers in local dev require Wrangler's `--test-scheduled` flag or manual HTTP triggers for testing.

---

## Secrets Management

### Production Secrets (Cloudflare)

All production secrets are stored as encrypted Cloudflare Worker secrets:

```bash
# Set a secret for a specific worker
wrangler secret put SUPABASE_SERVICE_KEY -c wrangler.toml

# List all secrets
wrangler secret list -c wrangler.toml

# Bulk update (from file)
wrangler secret:bulk .prod.vars -c wrangler.toml
```

### Secret Inventory

| Secret | Used By | Rotation Frequency |
|--------|---------|-------------------|
| `SUPABASE_URL` | All 4 | Never (stable URL) |
| `SUPABASE_SERVICE_KEY` | All 4 | On demand |
| `DATABASE_URL` | cf-astro, cf-email | On demand |
| `GEMINI_API_KEY` | cf-chatbot | On demand |
| `ANTHROPIC_API_KEY` | cf-chatbot | On demand |
| `RESEND_API_KEY` | cf-admin, cf-email | On demand |
| `ADMIN_API_KEY` | cf-chatbot | On rotation |
| `WHATSAPP_VERIFY_TOKEN` | cf-chatbot | On setup |
| `WHATSAPP_API_TOKEN` | cf-chatbot | 90 days (Meta) |
| `REVALIDATION_SECRET` | cf-astro, cf-admin | On rotation |
| `CF_ACCESS_AUD` | cf-admin | Never (stable) |
| `CF_TEAM_NAME` | cf-admin | Never (stable) |
| `UPSTASH_REDIS_REST_URL` | 3 services | Never (stable) |
| `UPSTASH_REDIS_REST_TOKEN` | 3 services | On demand |
| `BETTERSTACK_SOURCE_TOKEN` | cf-astro | On demand |
| `SENTRY_DSN` | 3 services | Never (stable) |
| `TURNSTILE_SECRET_KEY` | cf-astro | On demand |
| `SECURITY_ALERT_EMAIL` | cf-admin | Never |

### Secret Rotation Procedure
```bash
# 1. Generate new secret in provider dashboard
# 2. Update Cloudflare Worker secret
wrangler secret put RESEND_API_KEY -c wrangler.toml
# Enter new value when prompted

# 3. Redeploy affected workers
npm run cf:deploy

# 4. Verify operation
wrangler tail cf-astro --format json  # Watch for errors
```

---

## Deployment

### Individual Service Deployment

```bash
# cf-astro
cd cf-astro && npm run cf:deploy

# cf-admin
cd cf-admin && npm run cf:deploy

# cf-chatbot
cd cf-chatbot && npm run deploy

# cf-email-consumer
cd cf-email-consumer && npm run deploy
```

### Deployment Checklist
1. ✅ Run local tests
2. ✅ Check `.dev.vars` are NOT committed
3. ✅ Verify `wrangler.toml` environment config
4. ✅ Deploy to staging (if available)
5. ✅ Deploy to production
6. ✅ Monitor Sentry for new errors (5 min)
7. ✅ Verify critical paths (booking, login, chat)

### D1 Migrations

```bash
# Apply migration locally
wrangler d1 execute madagascar-db --local --file=migrations/001_initial.sql

# Apply migration to production
wrangler d1 execute madagascar-db --remote --file=migrations/001_initial.sql
```

### Supabase Migrations (Drizzle)

```bash
# Generate migration from schema changes
npx drizzle-kit generate

# Apply to Supabase
npx drizzle-kit push
```

---

## Useful Wrangler Commands

```bash
# Tail logs in real-time
wrangler tail cf-astro --format json
wrangler tail cf-chatbot

# List D1 databases
wrangler d1 list

# Query D1 remotely
wrangler d1 execute madagascar-db --remote --command "SELECT * FROM admin_feature_flags"

# List KV namespaces
wrangler kv namespace list

# Read a KV key
wrangler kv key get --namespace-id=xxx "session:abc-123"

# List R2 buckets
wrangler r2 bucket list

# List R2 objects
wrangler r2 object list madagascar-images

# Check worker status
wrangler deployments list -c wrangler.toml
```

---

## Troubleshooting

| Problem | Solution |
|---------|---------|
| `MODULE_NOT_FOUND` in local dev | Run `npm install` in the service directory |
| D1 binding not found | Check `wrangler.toml` has `[[d1_databases]]` section |
| KV returns `null` for existing key | KV is eventually consistent; wait 60s |
| Supabase connection timeout | Check `DATABASE_URL` includes `?sslmode=require` |
| Sentry not receiving events | Verify `SENTRY_DSN` and check sample rate |
| Turnstile validation fails locally | Use Turnstile test keys in `.dev.vars` |
| CF Access redirect loop | Ensure `CF_ACCESS_AUD` matches the Access application |
| Queue consumer not processing | Check queue binding name matches `wrangler.toml` |
