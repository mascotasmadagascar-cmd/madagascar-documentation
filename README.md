# Madagascar Pet Hotel — Platform Documentation

![Cloudflare Workers](https://img.shields.io/badge/Cloudflare_Workers-F38020?style=flat&logo=cloudflare&logoColor=white)
![Astro](https://img.shields.io/badge/Astro_6-BC52EE?style=flat&logo=astro&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-3ECF8E?style=flat&logo=supabase&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white)
![Resend](https://img.shields.io/badge/Resend-000000?style=flat&logo=resend&logoColor=white)
![Sentry](https://img.shields.io/badge/Sentry-362D59?style=flat&logo=sentry&logoColor=white)
![Upstash](https://img.shields.io/badge/Upstash_Redis-00E9A3?style=flat&logo=redis&logoColor=black)
![Cost](https://img.shields.io/badge/Infrastructure_Cost-$0%2Fmonth-brightgreen?style=flat)

> Complete technical and business documentation for Madagascar Pet Hotel's edge-native web platform — four Cloudflare Workers running at 330+ global PoPs, $0/month infrastructure cost, and full LFPDPPP compliance.

---

## Start Here

| You are… | Read this first |
|----------|----------------|
| **Business stakeholder / prospect** | [Executive Summary](./00-executive-summary.md) — ROI, $0/month story, compliance, service guarantees |
| **Onboarding developer** | [Developer Guide](./11-developer-guide.md) → [Architecture Overview](./01-architecture-overview.md) |
| **Security reviewer** | [Security Architecture](./05-security-architecture.md) → [Compliance & Privacy](./10-compliance-and-privacy.md) |
| **DevOps / infrastructure** | [Service Connections](./12-service-connections.md) → [Free Tier Master Reference](./13-free-tier-master-reference.md) |
| **AI / chatbot evaluator** | [cf-chatbot AI Pipeline](./cf-chatbot/ai-pipeline.md) → [Analytics & Costs](./cf-chatbot/analytics-and-costs.md) |
| **Future AI assistant (Claude / Codex)** | Read [Architecture Overview](./01-architecture-overview.md) + each project's `README.md` + `RULES.md` in the project root |

---

## Platform at a Glance

```
┌──────────────────────────────────────────────────────────────────┐
│                    Cloudflare Edge — 330+ PoPs                   │
├────────────┬─────────────┬──────────────┬────────────────────────┤
│  cf-astro  │  cf-admin   │  cf-chatbot  │  cf-email-consumer     │
│  Public    │  Admin      │  AI Chatbot  │  Queue Consumer        │
│  Website   │  Dashboard  │  WA + Web    │  Resend Dispatcher     │
├────────────┴─────────────┴──────────────┴────────────────────────┤
│         D1 · KV · R2 · Queues · Workers AI · Vectorize · AE     │
├──────────────────────────────────────────────────────────────────┤
│    Supabase PostgreSQL · Upstash Redis · Resend · Sentry         │
└──────────────────────────────────────────────────────────────────┘
```

| Service | Technology | Purpose |
|---------|-----------|---------|
| **cf-astro** | Astro 6 + Preact + Drizzle | Public website — bookings, SEO, i18n (es/en), ISR |
| **cf-admin** | Astro 6 + Preact SSR + CF Access | Staff portal — RBAC, CMS, audit log, diagnostics |
| **cf-chatbot** | Hono + Gemini + Claude | AI chatbot — WhatsApp + web widget, RAG pipeline |
| **cf-email-consumer** | Cloudflare Queue Consumer | Async email via Resend — booking confirmations, ARCO |

---

## System-Wide Documentation

| # | Document | Description |
|---|----------|-------------|
| 00 | [Executive Summary](./00-executive-summary.md) | Non-technical — business impact, $0/month story, compliance |
| 01 | [Architecture Overview](./01-architecture-overview.md) | System architecture, service map, inter-service communication |
| 02 | [Technology Stack](./02-technology-stack.md) | Every technology used, why chosen, alternatives considered |
| 03 | [Pricing & Cost Analysis](./03-pricing-and-cost-analysis.md) | Free tier utilization, scaling costs, platform comparison |
| 04 | [Performance Benchmarks](./04-performance-benchmarks.md) | Latency, throughput, cold starts, edge vs regional |
| 05 | [Security Architecture](./05-security-architecture.md) | Defense-in-depth, auth flows, secrets, security headers |
| 06 | [Industry Comparison](./06-industry-comparison.md) | Cloudflare vs AWS vs Vercel vs traditional hosting |
| 07 | [Scalability & Limits](./07-scalability-and-limits.md) | Hard limits, growth projections, horizontal scaling |
| 08 | [Observability & Monitoring](./08-observability-and-monitoring.md) | Sentry, Analytics Engine, logging strategy |
| 09 | [Data Architecture](./09-data-architecture.md) | D1, Supabase, KV, R2, Drizzle ORM, schema design |
| 10 | [Compliance & Privacy](./10-compliance-and-privacy.md) | LFPDPPP, ARCO, consent collection, data minimization |
| 11 | [Developer Guide](./11-developer-guide.md) | Setup, secrets management, deployment, local dev |
| 12 | [Service Connections](./12-service-connections.md) | Dependency map, shared infra IDs, request lifecycles |
| 13 | [Free Tier Master Reference](./13-free-tier-master-reference.md) | All service limits, daily usage, headroom %, upgrade triggers |
| — | [Implementations & Blockers](./implementations-and-blockers.md) | Active blockers — DLQ, Sentry gap, ESM migration |

---

## Project Documentation

### [cf-astro](./cf-astro/) — Public Website

Customer-facing website: bookings, SEO, bilingual (es/en), ISR caching, analytics.

| Document | Description |
|----------|-------------|
| [README](./cf-astro/README.md) | Project overview & quick reference |
| [Architecture](./cf-astro/architecture.md) | Internal architecture & middleware pipeline |
| [API Reference](./cf-astro/api-reference.md) | All REST API endpoints |
| [Database Schema](./cf-astro/database-schema.md) | Drizzle ORM schema & Supabase tables |
| [Middleware & Caching](./cf-astro/middleware-and-caching.md) | ISR cache, feature flags, analytics |
| [SEO & i18n](./cf-astro/seo-and-i18n.md) | Sitemaps, robots.txt, locale routing |
| [Bindings & Secrets](./cf-astro/bindings-and-secrets.md) | Wrangler config documentation |
| [Cost & Scaling](./cf-astro/cost-and-scaling.md) | Volume capacity & cost projections |

---

### [cf-admin](./cf-admin/) — Admin Dashboard

Internal portal: Cloudflare Zero Trust auth, 5-tier RBAC, CMS, booking management, chatbot admin, diagnostics.

| Document | Description |
|----------|-------------|
| [README](./cf-admin/README.md) | Project overview & quick reference |
| [Architecture](./cf-admin/architecture.md) | Middleware pipeline, 8-layer security |
| [Authentication](./cf-admin/authentication.md) | Zero Trust, KV sessions, RBAC, PLAC |
| [API Reference](./cf-admin/api-reference.md) | All admin API endpoints |
| [CMS & Revalidation](./cf-admin/cms-and-revalidation.md) | Content management, ISR webhooks, feature flags |
| [Audit & Diagnostics](./cf-admin/audit-and-diagnostics.md) | Ghost Audit Engine, login forensics, system health |
| [Bindings & Secrets](./cf-admin/bindings-and-secrets.md) | 15+ secrets documented with rotation policy |
| [Cost & Scaling](./cf-admin/cost-and-scaling.md) | Admin-specific capacity analysis |

---

### [cf-chatbot](./cf-chatbot/) — AI Chatbot

Dual-channel chatbot (WhatsApp + web widget): RAG pipeline, hybrid FTS5+Vectorize search, Gemini 2.0 Flash primary, Claude 3.5 Haiku fallback.

| Document | Description |
|----------|-------------|
| [README](./cf-chatbot/README.md) | Project overview & quick reference |
| [Architecture](./cf-chatbot/architecture.md) | Pipeline flow, module decomposition |
| [AI Pipeline](./cf-chatbot/ai-pipeline.md) | RAG, RRF hybrid search, multi-model fallback |
| [Channels & Conversations](./cf-chatbot/channels-and-conversations.md) | WhatsApp webhook, web widget, CORS, sessions, escalation |
| [Knowledge Base](./cf-chatbot/knowledge-base.md) | D1 schema, FTS5 index, Vectorize, admin CRUD |
| [Analytics & Costs](./cf-chatbot/analytics-and-costs.md) | Intent tracking, KB gaps, LLM cost per conversation |
| [Bindings & Secrets](./cf-chatbot/bindings-and-secrets.md) | 13 secrets documented with deployment commands |

---

### [cf-email-consumer](./cf-email-consumer/) — Email Queue Consumer

Cloudflare Queue consumer: async email delivery via Resend with Sentry distributed tracing across the queue boundary.

| Document | Description |
|----------|-------------|
| [README](./cf-email-consumer/README.md) | Project overview & quick reference |
| [Architecture](./cf-email-consumer/architecture.md) | Queue processing, DLQ (planned), Sentry tracing |
| [Email Templates](./cf-email-consumer/email-templates.md) | 4 email types — booking (admin + customer), contact, ARCO |
| [Bindings & Costs](./cf-email-consumer/bindings-and-costs.md) | Queue config, 3 secrets, Resend limits, $0/month breakdown |

---

## Key Numbers

| Metric | Value |
|--------|-------|
| Infrastructure cost | **$0 / month** (all free tiers) |
| Cloudflare Workers | 4 deployed (330+ PoPs) |
| Cold start latency | < 5 ms (V8 isolate model) |
| Edge response (p50) | < 50 ms |
| Emails / day (free) | 100 (Resend) — headroom ~70% at current volume |
| Daily Workers requests | < 0.01% of 100K free limit |
| First upgrade trigger | ~45 bookings/day → Resend Pro ($20/month) |
| LLM cost / conversation | ~$0.0013 (weighted avg, 95% Gemini 2.0 Flash) |
| Data protection law | LFPDPPP (Mexico) — ARCO rights implemented |

---

## Local Development

```bash
# Clone and start any service
cd cf-astro          && npm install && npm run dev   # → localhost:4321
cd cf-admin          && npm install && npm run dev   # → localhost:4322
cd cf-chatbot        && npm install && npm run dev   # → localhost:8787
cd cf-email-consumer && npm install && npm run dev   # → localhost:8788

# Deploy to production
cd cf-astro          && npm run cf:deploy
cd cf-admin          && npm run cf:deploy
cd cf-chatbot        && npm run deploy
cd cf-email-consumer && npm run deploy
```

See [Developer Guide](./11-developer-guide.md) for full secrets setup and Wrangler configuration.

---

## Overall Platform Score: 8.7 / 10

Full evaluation rationale in [Architecture Overview](./01-architecture-overview.md).
