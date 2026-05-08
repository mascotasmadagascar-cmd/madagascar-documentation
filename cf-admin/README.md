# cf-admin — Admin Dashboard

> Internal administration portal with Zero Trust authentication, RBAC, and CMS.

---

## Quick Reference

| Property | Value |
|----------|-------|
| **Framework** | Astro 6.1 + Preact islands |
| **Runtime** | Cloudflare Workers (V8 isolate) |
| **Adapter** | `@astrojs/cloudflare` |
| **Auth** | Cloudflare Zero Trust + KV sessions |
| **Domain** | `secure.madagascarhotelags.com` (planned) |
| **Port (dev)** | `localhost:4322` |
| **Deploy** | `npm run cf:deploy` |

## Purpose
Internal admin portal for Madagascar Pet Hotel staff and developers:
- 🔐 Zero Trust authentication (Google/GitHub IdP)
- 👥 User management with 4-tier RBAC (dev/admin/editor/viewer)
- 📋 Booking management (view, update status, notes)
- 📝 CMS with ISR revalidation webhook
- 🤖 Chatbot administration (KB management, config, analytics proxy)
- 📊 System diagnostics dashboard
- 📜 Audit command center with login security logs

## Key Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `astro` | ^6.1.2 | SSR framework |
| `@astrojs/cloudflare` | ^14.2.11 | CF Workers adapter |
| `preact` | ^10.29.0 | Interactive islands |
| `@supabase/supabase-js` | ^2.101.1 | Database client |
| `@sentry/astro` | ^10.51.0 | Error tracking |
| `@upstash/ratelimit` | ^2.0.8 | Rate limiting |
| `zod` | ^4.4.4 | Schema validation |
| `tailwindcss` | ^4.2.2 | Styling |
| `lucide-preact` | ^1.7.0 | Icons |

## File Structure
```
cf-admin/
├── src/
│   ├── pages/
│   │   ├── api/              # Admin API routes
│   │   │   ├── auth/         # Dev login, session management
│   │   │   ├── bookings/     # Booking CRUD
│   │   │   ├── chatbot/      # Chatbot admin proxy
│   │   │   ├── cms/          # Content management
│   │   │   ├── diagnostics/  # System health checks
│   │   │   └── users/        # User management
│   │   ├── dashboard/        # Dashboard pages
│   │   └── index.astro       # Login/landing page
│   ├── components/           # Preact islands
│   ├── layouts/              # Admin layout
│   ├── lib/
│   │   ├── auth/             # Session, RBAC, PLAC, Zero Trust
│   │   ├── csrf.ts           # CSRF token management
│   │   ├── env.ts            # Environment variable access
│   │   └── supabase.ts       # Supabase client factory
│   └── middleware.ts          # 8-layer security pipeline
├── wrangler.toml
└── package.json
```
