# cf-admin — Deploy & Operations

> Build pipeline, environment variables, and zero-downtime deployments.

---

## Deployment Pipeline

`cf-admin` uses Astro's Cloudflare adapter to build and deploy to Cloudflare Workers.

### Build Command
```bash
npm run build
```
Under the hood, Astro bundles the Preact islands, server-side API routes, and middleware into a single output directory (`dist/`) optimized for Cloudflare Workers.

### Deploy Command
```bash
npm run cf:deploy
```
This script executes `wrangler deploy` to push the built assets and worker code to Cloudflare.

---

## Environment Configuration

### Required Secrets

These secrets must be configured via `wrangler secret put` before deployment:

| Secret | Purpose |
|--------|---------|
| `SUPABASE_SERVICE_KEY` | Bypasses RLS for full admin database access |
| `RESEND_API_KEY` | Used to send security alerts |
| `CHATBOT_ADMIN_KEY` | Authenticates proxy requests to `cf-chatbot` |
| `REVALIDATION_SECRET` | Authenticates ISR revalidation webhook calls to `cf-astro` |
| `UPSTASH_REDIS_REST_TOKEN` | Required for Upstash rate limiting |
| `SENTRY_DSN` | Required for error tracking |
| `SENTRY_AUTH_TOKEN` | Required for source map upload during deployment |

### Public Environment Variables

Defined in `wrangler.toml` and injected at build/runtime:

```toml
[vars]
PUBLIC_SITE_URL = "https://secure.madagascarhotelags.com"
CF_ACCESS_AUD = "your-access-audience-tag"
CF_TEAM_NAME = "your-cloudflare-team-name"
SESSION_MAX_LIFETIME_MS = "86400000"
SECURITY_ALERT_EMAIL = "admin@example.com"
CHATBOT_WORKER_URL = "https://chatbot.madagascarhotelags.com"
UPSTASH_REDIS_REST_URL = "https://global-xxx.upstash.io"
```

---

## Local Development Lifecycle

1. **Install Dependencies**: `npm install`
2. **Setup Local Vars**: Copy `.dev.vars.example` to `.dev.vars` and populate with development secrets.
3. **Start Dev Server**: `npm run dev`
4. **Login**: Navigate to `http://localhost:4322`. Since Cloudflare Access is not available locally, you will see a specific development login screen. Enter `dev@example.com` to trigger the `api/auth/dev-login` bypass route.

---

## Zero-Downtime Deployments

Cloudflare Workers deployments are atomic. When you deploy:

1. **New Version Uploaded**: The new worker script and assets are uploaded.
2. **Traffic Cutover**: New incoming requests are routed to the new version.
3. **Old Version Drain**: Existing requests currently processing on the old version are allowed to finish.

**Session Persistence**:
Because sessions are stored externally in KV (`SESSION_KV`), deployments do not log users out. The new worker version seamlessly reads the existing session tokens.

**Database Schema Changes**:
If a deployment requires Supabase schema changes, always perform **additive** schema migrations first, deploy the code, and then perform destructive schema migrations later to ensure zero downtime.

---

## Access Control Setup (Cloudflare Zero Trust)

Before the dashboard can be used in production, the Cloudflare Access application must be configured:

1. **Create Application**: In the Cloudflare Dashboard → Zero Trust → Access → Applications.
2. **Define Domain**: Set the application to protect `secure.madagascarhotelags.com`.
3. **Identity Providers**: Enable Google Workspace or GitHub.
4. **Create Policy**: Allow access only to specific email addresses or groups (e.g., `*@madagascarhotelags.com`).
5. **Get Audience Tag**: Copy the "Audience Tag" (AUD) from the application settings.
6. **Configure Worker**: Set the `CF_ACCESS_AUD` variable in `wrangler.toml` to match this tag.

The middleware uses this `CF_ACCESS_AUD` tag to cryptographically verify that the JWT came from this specific Access application.
