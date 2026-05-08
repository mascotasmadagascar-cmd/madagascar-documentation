# 02 — Technology Stack

> Every technology used in the Madagascar Project, why it was chosen, and what alternatives were considered.

---

## Core Runtime

| Technology | Version | Purpose | Why Chosen |
|-----------|---------|---------|------------|
| **Cloudflare Workers** | V8 isolates | Serverless compute | $0 at small scale, <5ms cold start, 330+ global PoPs, native bindings to D1/KV/R2/Queues |
| **Astro 6** | ^6.1.2 | SSR framework | Island architecture (ships zero JS by default), native Cloudflare adapter, built-in i18n/content collections |
| **Preact** | ^10.29.0 | UI islands | 3KB alternative to React, full compat API, signals for reactive state |
| **TypeScript** | ^5.9.3 | Type safety | End-to-end type safety from schema to UI, Cloudflare workers-types |

### Why Not...
| Alternative | Reason Rejected |
|-------------|----------------|
| **Next.js** | Larger bundle, Vercel-optimized, higher cold starts on CF Workers |
| **React** | 40KB+ runtime for island components; Preact does the same at 3KB |
| **Remix** | Good edge support but Astro's zero-JS-by-default is better for content sites |
| **SvelteKit** | Viable alternative but team expertise favored Preact/JSX patterns |

---

## Styling

| Technology | Version | Purpose |
|-----------|---------|---------|
| **TailwindCSS 4** | ^4.2.2 | Utility-first CSS |
| **@tailwindcss/vite** | ^4.2.2 | Vite plugin for JIT compilation |
| **@tailwindcss/typography** | ^0.5.19 | Prose styling for CMS content |
| **class-variance-authority** | ^0.7.1 | Type-safe component variants |
| **clsx** | ^2.1.1 | Conditional class composition |
| **tailwind-merge** | ^3.5.0 | Deduplication of conflicting classes |

---

## Database & Storage

| Technology | Version | Purpose | Why Chosen |
|-----------|---------|---------|------------|
| **Supabase PostgreSQL** | Managed | Primary relational database | Full Postgres power, RLS, real-time, auth management, generous free tier (500MB) |
| **Drizzle ORM** | ^0.45.2 | Type-safe SQL (cf-astro, cf-email-consumer) | Tiny bundle (<50KB), no binary deps, native `postgres.js` driver for Workers |
| **postgres.js** | ^3.4.8 | PostgreSQL driver | Lightweight, works in Workers (no native bindings), connection pooling |
| **Cloudflare D1** | Binding | Edge SQLite database | $0 cost, <1ms latency (co-located with Worker), perfect for lookup tables and audit logs |
| **Cloudflare KV** | Binding | Key-value sessions/cache | Globally distributed, eventual consistency OK for sessions, 100K reads/day free |
| **Cloudflare R2** | Binding | Object storage (images, docs) | S3-compatible, $0 egress, 10GB free, private bucket support |
| **@supabase/supabase-js** | ^2.101.1 | Supabase client (cf-admin, cf-chatbot) | REST API for dashboard CRUD, real-time for admin |

### Why Not...
| Alternative | Reason Rejected |
|-------------|----------------|
| **Prisma** | 2MB+ engine binary, incompatible with Workers V8 isolates |
| **PlanetScale** | MySQL-based (Postgres preferred), higher cost at scale |
| **Turso (libSQL)** | Viable D1 alternative but adds external dependency |
| **Neon** | Excellent Postgres but Supabase's auth/RLS ecosystem is stronger |

---

## AI & Machine Learning

| Technology | Version | Purpose | Why Chosen |
|-----------|---------|---------|------------|
| **Workers AI** | Binding | Edge inference (FAQ gen, embeddings) | $0 (10K neurons/day), co-located with Worker, no network hop |
| **Vectorize** | Binding | Vector similarity search | Native CF binding, paired with Workers AI embeddings |
| **@cf/baai/bge-small-en-v1.5** | Workers AI | Embedding model | Free, fast, good quality for small KB |
| **Gemini API** | External | Primary LLM (chatbot) | Competitive pricing, thinking mode support, streaming |
| **Anthropic Claude API** | External | Fallback LLM (chatbot) | Higher quality reasoning, extended thinking support |
| **D1 FTS5** | Built-in | Full-text keyword search | SQLite native, $0, combined with Vectorize for hybrid search |

### Hybrid Search Strategy (RRF)
The chatbot uses **Reciprocal Rank Fusion** to combine two search modalities:
1. **Semantic**: Vectorize (dense embeddings)
2. **Keyword**: D1 FTS5 (sparse/exact match)

RRF formula: `score(d) = Σ 1/(k + rank_i(d))` where `k=60`

This outperforms either modality alone, especially for mixed language queries.

---

## Email & Communication

| Technology | Version | Purpose | Why Chosen |
|-----------|---------|---------|------------|
| **Resend** | REST API | Transactional email | Developer-friendly, 100 emails/day free, excellent deliverability |
| **Cloudflare Queues** | Binding | Async email decoupling | $0 (1M msgs/mo free), native DLQ support, built-in retry |
| **svix** | ^1.90.0 | Webhook signature verification | Resend uses Svix for HMAC-SHA256 delivery webhooks |
| **WhatsApp Cloud API** | External | WhatsApp messaging | Direct Meta API, no third-party middleware cost |

### Why Not...
| Alternative | Reason Rejected |
|-------------|----------------|
| **SendGrid** | More expensive, less developer-friendly API |
| **AWS SES** | $0.10/1K emails but requires AWS account management |
| **Twilio** | WhatsApp via Twilio adds unnecessary middleware cost |

---

## Security & Auth

| Technology | Version | Purpose |
|-----------|---------|---------|
| **Cloudflare Zero Trust** | Platform | Admin authentication (IdP gateway) |
| **Cloudflare Turnstile** | Platform | CAPTCHA replacement for public forms |
| **@upstash/ratelimit** | ^2.0.8 | Sliding window rate limiting |
| **@upstash/redis** | ^1.37.0 | Redis client for rate limit backend |
| **Zod** | ^3.25/^4.4 | Runtime schema validation |
| **crypto.subtle** | Web API | HMAC, JWT verification, CSRF tokens |

---

## Observability

| Technology | Version | Purpose |
|-----------|---------|---------|
| **@sentry/cloudflare** | ^10.51.0 | Error tracking + distributed tracing |
| **@sentry/astro** | ^10.51.0 | Astro-specific Sentry integration |
| **@sentry/browser** | ^10.49.0 | Client-side error reporting |
| **@sentry/vite-plugin** | ^5.2.0 | Source map upload at build |
| **@logtail/edge** | ^0.5.8 | BetterStack structured logging (cf-astro) |
| **Analytics Engine** | Binding | $0 edge telemetry & conversion tracking |
| **Cloudflare Observability** | Platform | Native logs + traces (all workers) |

---

## Build & Deploy

| Technology | Version | Purpose |
|-----------|---------|---------|
| **Wrangler** | ^4.79.0 | Cloudflare CLI (deploy, secrets, D1 migrations) |
| **Vite** | via Astro | Build tool, HMR, dev server |
| **dotenv-cli** | ^11.0.0 | Load `.dev.vars` for local development |
| **drizzle-kit** | ^0.31.10 | Schema migrations for Supabase |

---

## Utility Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| **lucide-preact** | ^1.7.0 | Icon library (tree-shakeable SVG icons) |
| **date-fns** | ^4.1.0 | Date manipulation |
| **date-fns-tz** | ^3.2.0 | Timezone handling (Mexico City) |
| **embla-carousel** | ^8.6.0 | Carousel component |
| **@preact/signals** | ^2.9.0 | Reactive state management |
