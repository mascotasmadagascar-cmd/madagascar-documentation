# cf-admin — Admin Portal

Internal administration portal for Madagascar Pet Hotel. Secured with Cloudflare Zero Trust, 5-tier RBAC, page-level access control, ghost audit logging, and a full diagnostics suite.

---

## Quick Reference

| Property | Value |
|----------|-------|
| Domain | `secure.madagascarhotelags.com` |
| Framework | Astro 6 + Preact islands |
| Runtime | Cloudflare Workers (V8 isolate) |
| Auth | Cloudflare Zero Trust (Google / GitHub / OTP IdP) |
| Database | Cloudflare D1 (`madagascar-db`) + Supabase PostgreSQL |
| Port (Astro dev) | `4322` |
| Port (Wrangler dev) | `3000` |
| Deploy | `npm run cf:deploy` |
| Cron | `*/5 * * * *` (288 invocations/day) |

---

## Purpose

cf-admin is the secure operations hub for hotel staff and developers:

- Zero Trust authentication with full JWT re-verification (Google / GitHub / OTP)
- 5-tier role hierarchy: dev / owner / super_admin / admin / staff
- Page-Level Access Control (PLAC) with per-user overrides and deny-wins resolution
- Ghost Audit Engine — every action logged async via `ctx.waitUntil()` (zero latency)
- Login forensics with Tier-1 Cloudflare geo/TLS/bot data and security alert emails
- Booking management with Supabase source of truth + D1 shadow operational state
- CMS with ISR revalidation to cf-astro via ASTRO_SERVICE service binding
- Chatbot administration proxied to cf-chatbot via CHATBOT_SERVICE service binding
- Diagnostics portal: connectivity, functional, and security test suites
- 8-layer middleware security pipeline (469 lines)
- Rate limiting via Upstash Redis sliding window
- 20 D1 migrations tracked

---

## Security at a Glance

| Layer | Mechanism |
|-------|-----------|
| Network | Cloudflare Zero Trust (before Worker runs) |
| Identity | RS256 JWT re-verification against JWKS (`mascotas.cloudflareaccess.com`) |
| Session | KV-backed, 24-hour hard expiry, 30-minute role freshness re-check |
| Authorization | 5-tier RBAC + PLAC hashmap lookup (O(1) per request) |
| Mutations | CSRF Origin/Referer validation on all POST/PUT/PATCH/DELETE |
| Rate limiting | Upstash Redis sliding window (per endpoint) |
| Audit | Ghost Audit Engine — INSERT-only, async, privacy-safe IP hashing |
| Revocation | 3-layer force-logout: KV delete + revocation flag + CF API session revoke |
| Headers | HSTS, CSP, X-Frame-Options DENY, X-Content-Type-Options nosniff |
| Monitoring | Sentry error tracking + distributed traces |

---

## Tech Stack

| Package | Purpose |
|---------|---------|
| `astro` ^6 | SSR framework |
| `@astrojs/cloudflare` | CF Workers adapter |
| `preact` ^10 | Interactive islands |
| `@supabase/supabase-js` ^2 | Supabase service-role client |
| `@sentry/astro` ^10 | Error tracking + distributed tracing |
| `@upstash/ratelimit` ^2 | Redis sliding-window rate limiting |
| `zod` ^4 | Schema validation |
| `tailwindcss` ^4 | Styling |
| `lucide-preact` | Icons |

---

## Directory Structure

```
cf-admin/
├── src/
│   ├── middleware.ts                 # 8-layer security pipeline (469 lines)
│   ├── workers/
│   │   ├── cf-entry.ts              # Worker entrypoint (cron + Astro server)
│   │   └── scheduled-log-sync.ts    # CF Access audit log poller
│   ├── pages/
│   │   ├── api/                     # 45 API endpoints
│   │   ├── dashboard/               # Admin UI pages
│   │   ├── index.astro              # Public landing
│   │   ├── privacy.astro
│   │   └── terms.astro
│   ├── components/
│   │   ├── admin/                   # Preact islands (users, content, logs, settings,
│   │   │                            #   bookings, chatbot, debug)
│   │   ├── auth/SessionWatchdog     # Session expiry monitor
│   │   ├── dashboard/               # Dashboard widgets
│   │   └── navigation/              # Sidebar, TopBar, CommandPalette, ThemeToggle
│   ├── layouts/
│   │   ├── AdminLayout.astro
│   │   └── ChatbotLayout.astro
│   └── lib/
│       ├── auth/
│       │   ├── session.ts           # KV session management (254 lines)
│       │   ├── rbac.ts              # 5-tier role hierarchy (105 lines)
│       │   ├── plac.ts              # Page-level access control (390 lines)
│       │   ├── cloudflare-access.ts # CF Zero Trust JWT verification (180 lines)
│       │   ├── security-logging.ts  # Login forensics + email alerts (344 lines)
│       │   └── guard.ts             # requireAuth() helper
│       ├── audit.ts                 # Ghost Audit Engine (134 lines)
│       ├── ratelimit.ts             # Upstash Redis rate limiting (62 lines)
│       ├── cms.ts                   # CMS I/O + R2 + ISR revalidation (302 lines)
│       ├── supabase.ts              # Supabase admin client (service role)
│       ├── env.ts                   # CF Workers env accessor
│       ├── api.ts                   # Response helpers (jsonOk, jsonError, withETag)
│       ├── csrf.ts                  # CSRF token validation
│       └── diagnostics/
│           ├── runner.ts            # Orchestrates diagnostic test suite
│           ├── benchmarks.ts        # Performance metrics
│           └── tests/               # Connectivity, security, functional tests
├── migrations/                      # 20 D1 migrations
├── email-templates/                 # Resend HTML email templates
├── astro.config.ts
├── wrangler.toml                    # All CF bindings + cron
└── sentry.*.config.ts
```

---

## Cloudflare Bindings

| Binding | Type | Name / ID | Purpose |
|---------|------|-----------|---------|
| `DB` | D1 Database | `madagascar-db` (`7fca2a07-d7b4-449d-b446-408f9187d3ca`) | Audit logs, PLAC config, CMS blocks, login forensics, booking shadow state |
| `IMAGES` | R2 Bucket | `madagascar-images` | Gallery + hero images (CDN: `cdn.madagascarhotelags.com`) |
| `SESSION` | KV Namespace | cf-admin session KV | KV sessions, revocation flags, cron timestamp |
| `EMAIL_QUEUE` | Queue Producer | `madagascar-emails` | Async email via Resend |
| `ANALYTICS` | Analytics Engine | `madagascar_analytics` | Optional admin usage metrics |
| `CHATBOT_SERVICE` | Service Binding | cf-chatbot worker | Zero-latency internal chatbot admin proxy |
| `ASTRO_SERVICE` | Service Binding | cf-astro worker | Zero-latency ISR revalidation |

---

## Key Numbers

| Metric | Value |
|--------|-------|
| Middleware layers | 8 |
| API endpoints | 45 |
| D1 migrations | 20 |
| Cron invocations/day | 288 |
| Session hard expiry | 24 hours |
| Role re-check interval | 30 minutes |
| PLAC map refresh interval | 60 minutes |
| RBAC tiers | 5 (dev / owner / super_admin / admin / staff) |
| Monthly cost | $0 |

---

## Local Development

```bash
# Install dependencies
npm install

# Copy and populate local secrets
cp .dev.vars.example .dev.vars
# Edit .dev.vars with your local secrets

# Start Astro dev server (port 4322)
npm run dev

# Start Wrangler dev server (port 3000, full CF bindings)
npm run cf:dev

# Run D1 migrations locally
wrangler d1 migrations apply madagascar-db --local

# Deploy to production
npm run cf:deploy
```

Local dev automatically uses `LOCAL_DEV_ADMIN_EMAIL` (`harshil.8136@gmail.com`) as the session identity — no CF Zero Trust required. `isLocalDev()` checks whether `SITE_URL` contains `localhost`. If `SITE_URL` is missing, the system defaults to **production mode** (fail-secure, not fail-open).

---

## CF Zero Trust Setup (One-Time)

1. Cloudflare Dashboard → Zero Trust → Access → Applications
2. Protect `secure.madagascarhotelags.com`
3. Identity providers: Google / GitHub / OTP
4. Copy the Audience Tag → set as `CF_ACCESS_AUD` in `wrangler.toml [vars]`
5. Team name: `mascotas` (already set in `[vars]`)

---

## Related Services

| Service | Domain | Relationship |
|---------|--------|-------------|
| cf-astro | `madagascarhotelags.com` | Public site — receives ISR revalidation via ASTRO_SERVICE binding |
| cf-chatbot | `charlar.madagascarhotelags.com` | Chatbot — proxied via CHATBOT_SERVICE binding |
| cf-email-consumer | (queue consumer) | Receives emails queued via EMAIL_QUEUE |
