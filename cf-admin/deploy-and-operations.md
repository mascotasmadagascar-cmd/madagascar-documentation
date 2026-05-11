# cf-admin — Deploy & Operations

Build pipeline, environment setup, deployment commands, and CF Zero Trust configuration.

---

## Build & Deploy Commands

```bash
# Install dependencies
npm install

# Build (Astro SSR output → dist/)
npm run build

# Deploy to Cloudflare Workers
npm run cf:deploy
# Equivalent to: astro build && wrangler deploy

# Run D1 migrations against production
wrangler d1 migrations apply madagascar-db --remote

# Run D1 migrations locally
wrangler d1 migrations apply madagascar-db --local

# Verify all secrets are set
wrangler secret list
```

---

## Local Development Lifecycle

1. **Install dependencies:** `npm install`
2. **Set up local secrets:** Copy `.dev.vars.example` to `.dev.vars` and populate (see [bindings-and-secrets.md](./bindings-and-secrets.md))
3. **Start Astro dev server (port 4322):** `npm run dev`
4. **Start Wrangler dev server (port 3000, full CF bindings):** `npm run cf:dev`
5. **Navigate to** `http://localhost:4322/` — the public landing page
6. **Log in:** Visit `http://localhost:4322/api/auth/dev-login` — since CF Zero Trust is not available locally, this bootstraps a dev session for `LOCAL_DEV_ADMIN_EMAIL`

Local dev auto-uses `LOCAL_DEV_ADMIN_EMAIL` (`harshil.8136@gmail.com` from `wrangler.toml [vars]`). No CF Zero Trust required. No JWT verification. Role defaults to `dev` for full local access.

**Fail-secure:** `isLocalDev()` checks whether `SITE_URL` contains `localhost`. If `SITE_URL` is missing, it defaults to production mode — the dev bypass never activates accidentally in production.

---

## Setting Production Secrets

All secrets must be set before the first deployment. See the full list in [bindings-and-secrets.md](./bindings-and-secrets.md).

```bash
# Core secrets
wrangler secret put SUPABASE_SERVICE_ROLE_KEY
wrangler secret put REVALIDATION_SECRET      # Must match cf-astro's value
wrangler secret put IP_HASH_SECRET
wrangler secret put RESEND_API_KEY

# Upstash Redis
wrangler secret put UPSTASH_REDIS_REST_URL
wrangler secret put UPSTASH_REDIS_REST_TOKEN

# Cloudflare API
wrangler secret put CLOUDFLARE_API_TOKEN
wrangler secret put CLOUDFLARE_ZONE_ID
wrangler secret put CF_API_TOKEN_READ_LOGS
wrangler secret put CF_API_TOKEN_ZT_WRITE

# Chatbot integration (must match cf-chatbot's ADMIN_API_KEY)
wrangler secret put CHATBOT_ADMIN_API_KEY
wrangler secret put CHATBOT_WORKER_URL

# Sentry (build-time only, CI secret)
wrangler secret put SENTRY_AUTH_TOKEN
wrangler secret put SENTRY_ORG_SLUG
wrangler secret put SENTRY_PROJECT_SLUG
```

---

## CF Zero Trust Setup (One-Time)

Before the portal is accessible in production:

1. Cloudflare Dashboard → Zero Trust → Access → Applications
2. Create application protecting `secure.madagascarhotelags.com`
3. Enable identity providers: Google / GitHub / OTP
4. Set policy (email allowlist or group-based)
5. Copy the **Audience Tag** from the application settings
6. Set `CF_ACCESS_AUD` in `wrangler.toml [vars]` to match the audience tag
7. Confirm `CF_TEAM_NAME = "mascotas"` in `[vars]` matches your Zero Trust team subdomain

The middleware verifies the `aud` claim in every CF Access JWT against `CF_ACCESS_AUD`. If there is a mismatch, all logins are rejected.

---

## D1 Migrations

20 migrations in `migrations/` directory. Always apply migrations before deploying code that depends on schema changes.

```bash
# Production
wrangler d1 migrations apply madagascar-db --remote

# Local
wrangler d1 migrations apply madagascar-db --local

# Check migration status
wrangler d1 migrations list madagascar-db --remote
```

Migration list: see [architecture.md](./architecture.md) for the full table of all 20 migrations and their purposes.

---

## Zero-Downtime Deployments

Cloudflare Workers deployments are atomic:

1. New Worker script and assets are uploaded
2. New incoming requests are routed to the new version immediately
3. In-flight requests on the old version are allowed to complete

**Session persistence:** Sessions are stored in KV (`SESSION` namespace), not in Worker memory. Deployments do not log users out. The new Worker version reads existing session cookies seamlessly.

**Schema changes:** Always apply additive D1 migrations before deploying code that reads new columns. Apply destructive migrations (column drops, constraint changes) after deployment when the old code is no longer running.

---

## Environment Variable Reference

| Variable | Type | Source | Value |
|----------|------|--------|-------|
| `SITE_URL` | var | wrangler.toml | `https://secure.madagascarhotelags.com` |
| `PUBLIC_ASTRO_URL` | var | wrangler.toml | `https://madagascarhotelags.com` |
| `PUBLIC_SUPABASE_URL` | var | wrangler.toml | `https://zlvmrepvypucvbyfbpjj.supabase.co` |
| `PUBLIC_CDN_URL` | var | wrangler.toml | `https://cdn.madagascarhotelags.com` |
| `CF_TEAM_NAME` | var | wrangler.toml | `mascotas` |
| `CF_ACCESS_AUD` | var | wrangler.toml | `680d415033...` (from CF dashboard) |
| `CF_ACCOUNT_ID` | var | wrangler.toml | `320d1ebab5143958d2acd481ea465f52` |
| `ADMIN_EMAIL` | var | wrangler.toml | `mascotasmadagascar@gmail.com` |
| `SESSION_MAX_LIFETIME_MS` | var | wrangler.toml | `86400000` (24 hours) |
| `SESSION_REFRESH_INTERVAL_MS` | var | wrangler.toml | `1800000` (30 minutes) |
| `LOCAL_DEV_ADMIN_EMAIL` | var | wrangler.toml | `harshil.8136@gmail.com` |
| `SUPABASE_SERVICE_ROLE_KEY` | secret | wrangler secret | Service-role JWT |
| `REVALIDATION_SECRET` | secret | wrangler secret | Must match cf-astro |
| `CHATBOT_ADMIN_API_KEY` | secret | wrangler secret | Must match cf-chatbot `ADMIN_API_KEY` |

Full secret list: see [bindings-and-secrets.md](./bindings-and-secrets.md).

---

## Monitoring After Deployment

1. **Sentry:** Check the Sentry project for any unhandled exceptions in the first 15 minutes post-deploy
2. **`GET /api/health`:** Public endpoint — verifies D1, KV, R2, Supabase connectivity
3. **`GET /api/diagnostics/ping`:** Simple liveness check
4. **CF Dashboard → Workers → cf-admin-madagascar:** Review error rate and CPU time metrics
5. **Cron:** First cron invocation occurs within 5 minutes of deploy — check CF dashboard → Workers → Cron Triggers for success/failure status

---

## Rollback

Cloudflare Workers rollback via the dashboard:

1. CF Dashboard → Workers & Pages → cf-admin-madagascar → Deployments
2. Find the previous stable deployment
3. Click "Rollback to this deployment"

Or via CLI:
```bash
# List deployments
wrangler deployments list

# Rollback to specific deployment ID
wrangler rollback <deployment-id>
```

Rollback does not affect KV sessions, D1 data, or R2 objects — only the Worker code is rolled back.
