# cf-astro — Public Website

Customer-facing website for Madagascar Pet Hotel (madagascarhotelags.com), built on Astro 6 + Preact islands and deployed as a Cloudflare Worker.

---

## Project Identity

| Property | Value |
|----------|-------|
| Domain | `madagascarhotelags.com` |
| Framework | Astro 6 + Preact islands |
| Styling | Tailwind CSS v4 |
| Adapter | `@astrojs/cloudflare` (passthroughImageService) |
| Output mode | `static` (SSR opt-in per route via `export const prerender = false`) |
| Dev port | `4321` |
| Build ID | `Date.now().toString(36)` injected as `__BUILD_ID__` via Vite define |
| Last updated | `__LAST_UPDATED__` (ISO timestamp injected at build) |
| Default locale | `es` (Spanish) |
| Admin email | `booking@madagascarhotelags.com` |

---

## Key Features

| Feature | Implementation |
|---------|---------------|
| ISR (Incremental Static Regeneration) | KV-backed HTML cache with 24h TTL and deploy-scoped keys |
| Bilingual routing (es/en) | Astro i18n with `prefixDefaultLocale: true` |
| Online booking wizard | Multi-step Preact island (BookingWizard.tsx), 3-phase DB insert |
| ARCO legal request form | ArcoForm.tsx with 6-layer file validation, R2 storage |
| Cookie consent | ConsentBanner.tsx with checkbox fingerprinting and granular accept/reject |
| MCP server | JSON-RPC 2.0 endpoint at `/api/mcp` for Claude/AI integrations |
| Workers AI | `@cf/meta/llama-3-8b-instruct` for blog post and FAQ generation |
| PostHog proxy | `/api/ingest/[...path]` — no direct client→posthog calls |
| IndexNow | Bing + Yandex crawl acceleration on every cache revalidation |
| Cloudflare Turnstile | CAPTCHA on ARCO document upload |
| Rate limiting | Upstash Redis sliding window + in-memory Map fallback |
| Structured data (JSON-LD) | Organization, LocalBusiness, Article, Service, BreadcrumbList |
| Feature flags | 3-tier cache (memory 10s → CF Cache API 60s → D1) |
| Resend email webhooks | Svix HMAC-SHA256 delivery event tracking |
| Analytics Engine | Edge telemetry at $0: page views, booking events, CTA events |
| Sentry error tracking | `@sentry/cloudflare`, source maps uploaded and deleted at build |
| BetterStack logging | `@logtail/edge`, batch 10, flush 1000ms |
| Security headers | CSRF origin check, timing-safe comparisons, HTML sanitization |
| Image gallery | Embla carousel (InfiniteGalleryIsland) + LightboxIsland |

---

## Tech Stack

| Package | Purpose |
|---------|---------|
| `astro` | SSR framework and build system |
| `@astrojs/cloudflare` | Cloudflare Workers adapter |
| `preact` | Interactive island components |
| `drizzle-orm` | Type-safe SQL ORM |
| `postgres` (postgres.js) | PostgreSQL driver (`max:1`, `idle_timeout:20`) |
| `tailwindcss` v4 | Utility-first CSS |
| `zod` | Request body validation |
| `@upstash/ratelimit` | Sliding-window rate limiting |
| `@logtail/edge` | BetterStack structured logging |
| `@sentry/cloudflare` | Error tracking and source maps |
| `@sentry/vite-plugin` | CI source map upload |

---

## Directory Structure

```
cf-astro/
├── db/migrations/
│   ├── 0001_initial_schema.sql    # bookings, pets, consent, privacy
│   ├── 0002_cms_content.sql       # CMS content table
│   ├── 0003_seed_services_pricing.sql
│   └── 0004_contact_messages.sql
├── src/
│   ├── components/
│   │   ├── booking/               # BookingWizard.tsx, DateRangePicker.tsx, FormInput.tsx,
│   │   │                          # PetTypeSelector.tsx, ServiceSelector.tsx, StepIndicator.tsx,
│   │   │                          # BookingNavbar.astro
│   │   ├── forms/ArcoForm.tsx     # ARCO legal request form (Preact)
│   │   ├── islands/               # ConsentBanner.tsx, AutoTabs.tsx,
│   │   │                          # InfiniteGalleryIsland.tsx, LightboxIsland.tsx
│   │   ├── layout/                # Header.astro, Footer.astro
│   │   ├── sections/              # Hero, Services, Gallery, FAQ, Testimonials, About, Contact
│   │   ├── seo/                   # SchemaMarkup.astro, BlogPostSchema.astro, ServicePageSchema.astro
│   │   └── ui/                    # Button.astro, Card.astro, FloatingWhatsApp.astro
│   ├── layouts/                   # BaseLayout.astro, BookingLayout.astro, MarketingLayout.astro
│   ├── pages/
│   │   ├── api/                   # All API endpoints (see table below)
│   │   ├── es/                    # index, booking, services, franchise, blog/[slug],
│   │   │                          # legal/arco, privacy, terms
│   │   └── en/                    # Same structure in English
│   ├── lib/
│   │   ├── db/client.ts           # Drizzle ORM + postgres.js
│   │   ├── db/schema.ts           # All table definitions
│   │   ├── consent/               # consent-fingerprint.ts, consent-store.ts
│   │   ├── email/queue-types.ts   # Queue message contracts
│   │   ├── cms.ts                 # CMS content retrieval from KV/D1
│   │   ├── env.ts                 # CF Workers env accessor
│   │   ├── error-context.ts       # Sentry captureApiError()
│   │   ├── logger.ts              # BetterStack Logtail
│   │   ├── rate-limit.ts          # Upstash Redis + in-memory fallback
│   │   ├── security.ts            # assertOrigin(), timingSafeEq(), sanitizeHtml()
│   │   ├── turnstile.ts           # Cloudflare Turnstile verifyTurnstile()
│   │   ├── pricing.ts             # Service pricing lookup
│   │   ├── indexnow.ts            # Bing/Yandex IndexNow pings
│   │   └── schemas/booking.ts     # Zod validation schemas
│   ├── i18n/config.ts             # Locale routing + t() helper
│   ├── i18n/translations/es.json  # Spanish messages
│   ├── i18n/translations/en.json  # English messages
│   └── middleware.ts              # ISR + feature flags + page view analytics
├── public/                        # Static assets (robots.txt, favicons, etc.)
├── wrangler.toml                  # Cloudflare configuration
├── astro.config.ts                # Astro configuration
└── package.json
```

---

## API Endpoints Summary

| Method | Path | Auth | Rate Limit | Purpose |
|--------|------|------|------------|---------|
| POST | `/api/booking` | CSRF origin check | 5/60s/IP | Submit pet hotel booking |
| POST | `/api/contact` | CSRF origin check | 3/60s/IP | Submit contact form |
| POST | `/api/consent` | None | 5/60s/IP | Record cookie/booking consent |
| POST | `/api/privacy/arco` | CSRF origin check | 3/60s/IP | Submit ARCO data rights request |
| POST | `/api/arco/submit` | CSRF + Turnstile CAPTCHA | 3/60s/IP | Upload identity document for ARCO |
| GET | `/api/arco/get-document` | `X-Admin-Secret` header | None | Download ARCO identity document (admin) |
| GET | `/api/health` | `Authorization: Bearer {HEALTH_CHECK_SECRET}` | None | D1 + KV health check |
| POST | `/api/revalidate` | `Authorization: Bearer {REVALIDATION_SECRET}` | None | Purge ISR cache + write CMS data |
| POST | `/api/analytics/track` | None | 60/60s/IP | Track CTA/funnel events |
| GET | `/api/analytics/summary` | `Authorization: Bearer {REVALIDATION_SECRET}` | None | Weekly analytics digest |
| POST | `/api/admin/generate-blog-draft` | `Authorization: Bearer {ADMIN_AI_SECRET}` | None | AI-generate blog post via Workers AI |
| POST | `/api/admin/generate-faqs` | `Authorization: Bearer {ADMIN_AI_SECRET}` | None | AI-generate 12 FAQ Q&As |
| POST | `/api/webhooks/resend` | Svix HMAC-SHA256 | None | Resend email delivery events |
| GET | `/api/media/[...path]` | None (dev-only) | None | R2 image proxy (dev only) |
| GET+POST | `/api/ingest/[...path]` | None | None | PostHog analytics proxy |
| POST | `/api/mcp` | None | None | JSON-RPC 2.0 MCP server for AI integrations |
| GET | `/api/test-services` | `Authorization: Bearer {HEALTH_CHECK_SECRET}` | None | Analytics Engine smoke test |

---

## Cloudflare Bindings

| Binding | Type | Name / ID | Purpose |
|---------|------|-----------|---------|
| `DB` | D1 Database | `madagascar-db` (`7fca2a07-d7b4-449d-b446-408f9187d3ca`) | Feature flags, CMS, ARCO legal records |
| `ISR_CACHE` | KV Namespace | `cf-astro-isr-cache` (`d9cea8c7e20f4b328b8cb3b04104138c`) | Page HTML cache (24h TTL) |
| `SESSION` | KV Namespace | `cf-astro-session` (`bee123e795504473accf58ac5b6de13d`) | Astro 6 adapter session storage |
| `IMAGES` | R2 Bucket | `madagascar-images` | CMS/gallery images (shared with cf-admin) |
| `ARCO_DOCS` | R2 Bucket | `arco-documents` | Private identity docs from ARCO requests |
| `EMAIL_QUEUE` | Queue Producer | `madagascar-emails` | Async email dispatch |
| `ANALYTICS` | Analytics Engine | `madagascar_analytics` | Edge telemetry ($0) |
| `AI` | Workers AI | — | `@cf/meta/llama-3-8b-instruct` blog/FAQ generation |

---

## Key Numbers

| Metric | Value |
|--------|-------|
| Monthly cost (10–20 bookings/day) | $0 |
| ISR cache TTL | 24 hours |
| Feature flag memory TTL | 10 seconds |
| Feature flag CF Cache TTL | 60 seconds |
| DB connections per isolate | 1 (`max: 1`) |
| DB idle timeout | 20 seconds |
| Workers AI quota | 10,000 neurons/day (free) |
| KV write headroom (first bottleneck) | ~20x at current scale |
| Rate limit — booking | 5 requests / 60s / IP |
| Rate limit — contact | 3 requests / 60s / IP |
| Rate limit — analytics/track | 60 requests / 60s / IP |
| ARCO file size limit | 5 MB |
| ARCO allowed extensions | jpg, jpeg, png, webp |
| Booking reference format | `MAD-XXXXXXXXXX` |
| ARCO ticket format | `ARCO-XXXXXX` |
| Email audit log — status lifecycle | queued → sent_to_provider → delivered / bounced / opened / clicked / complained / delayed / failed |
| Resend webhook replay protection | 5-minute timestamp tolerance |

---

## Local Development

```bash
# Install dependencies
npm install

# Start dev server
npm run dev
# Runs on http://localhost:4321

# Type-check
npm run astro check

# Build for production
npm run build

# Preview production build locally
npm run preview
```

### .dev.vars (required for local dev)
```ini
DATABASE_URL=postgresql://cf_astro_writer:...@db.zlvmrepvypucvbyfbpjj.supabase.co:5432/postgres
REVALIDATION_SECRET=dev-revalidation-secret
HEALTH_CHECK_SECRET=dev-health-secret
ADMIN_AI_SECRET=dev-ai-secret
ARCO_ADMIN_SECRET=dev-arco-secret
TURNSTILE_SECRET_KEY=1x0000000000000000000000000000000AA  # always-pass test key
UPSTASH_REDIS_REST_TOKEN=...
BETTERSTACK_SOURCE_TOKEN=...
RESEND_API_KEY=...
RESEND_WEBHOOK_SECRET=...
```

---

## Deploy Commands

```bash
# Deploy to production
npx wrangler deploy

# Deploy to preview environment
npx wrangler deploy --env preview

# Set a secret
npx wrangler secret put DATABASE_URL

# Run D1 migration
npx wrangler d1 execute madagascar-db --remote --file=db/migrations/0001_initial_schema.sql

# Tail live logs
npx wrangler tail

# Query D1
npx wrangler d1 execute madagascar-db --remote --command "SELECT * FROM admin_feature_flags"
```

---

## Related Projects

| Project | Role |
|---------|------|
| `cf-admin` | Admin portal; triggers `/api/revalidate` on CMS changes; shares `IMAGES` R2 bucket and `REVALIDATION_SECRET` |
| `cf-email-consumer` | Queue consumer that processes `EMAIL_QUEUE` messages via Resend; shares `DATABASE_URL` (`cf_astro_writer` role) |
| `cf-chatbot` | Customer chat widget; shares `UPSTASH_REDIS_REST_TOKEN` (different key prefix) |

---

See [architecture.md](./architecture.md) for the full request lifecycle, middleware pipeline, and data flow diagrams.
