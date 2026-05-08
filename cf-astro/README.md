# cf-astro — Public Website

> Customer-facing website for Madagascar Pet Hotel, built on Astro 6 + Preact.

---

## Quick Reference

| Property | Value |
|----------|-------|
| **Framework** | Astro 6.1 + Preact islands |
| **Runtime** | Cloudflare Workers (V8 isolate) |
| **Adapter** | `@astrojs/cloudflare` |
| **Styling** | TailwindCSS 4 + custom design tokens |
| **Domain** | `madagascarhotelags.com` |
| **Port (dev)** | `localhost:4321` |
| **Deploy** | `npm run cf:deploy` |

## Purpose
The public-facing website for Madagascar Pet Hotel in Aguascalientes, Mexico. Handles:
- 🏠 Landing page with service information
- 📅 Online booking with pet information
- 📧 Contact form with email dispatch
- 📋 ARCO rights request form
- 🌐 Bilingual content (Spanish primary, English secondary)
- 🔍 SEO optimization with sitemaps and structured data
- ⚡ ISR caching for sub-10ms page loads

## Key Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `astro` | ^6.1.2 | SSR framework |
| `@astrojs/cloudflare` | ^14.2.11 | CF Workers adapter |
| `preact` | ^10.29.0 | Interactive islands |
| `drizzle-orm` | ^0.45.2 | Type-safe SQL |
| `postgres` | ^3.4.8 | PostgreSQL driver |
| `@sentry/astro` | ^10.51.0 | Error tracking |
| `@logtail/edge` | ^0.5.8 | Structured logging |
| `@upstash/ratelimit` | ^2.0.8 | Rate limiting |
| `zod` | ^3.25.48 | Schema validation |
| `tailwindcss` | ^4.2.2 | Styling |

## Architecture
See [architecture.md](./architecture.md) for detailed internals.

## File Structure
```
cf-astro/
├── src/
│   ├── pages/           # Astro pages (file-based routing)
│   │   ├── api/         # REST API endpoints
│   │   ├── en/          # English locale pages
│   │   └── index.astro  # Spanish homepage
│   ├── components/      # Preact islands + Astro components
│   ├── layouts/         # Page layouts
│   ├── lib/             # Business logic, DB, utils
│   │   ├── db/          # Drizzle schema + queries
│   │   └── logger.ts    # BetterStack logger
│   ├── middleware.ts     # ISR cache + feature flags + analytics
│   └── env.d.ts         # Type declarations
├── public/              # Static assets
├── wrangler.toml        # Cloudflare configuration
├── astro.config.ts      # Astro configuration
└── package.json
```
