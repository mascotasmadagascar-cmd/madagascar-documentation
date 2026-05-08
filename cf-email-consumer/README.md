# cf-email-consumer — Email Microservice

> Cloudflare Queue consumer for transactional email dispatch and error handling.

---

## Quick Reference

| Property | Value |
|----------|-------|
| **Framework** | Vanilla TypeScript |
| **Runtime** | Cloudflare Workers (V8 isolate) |
| **Trigger** | Cloudflare Queues (`madagascar-emails`) |
| **Email API** | Resend |
| **Database** | Supabase (via Drizzle ORM) |
| **Error Tracking**| Sentry |
| **Deploy** | `npm run deploy` |

## Purpose
An asynchronous worker dedicated solely to processing queue messages and dispatching transactional emails. 
- Decouples slow, third-party API calls (Resend) from the main request path of `cf-astro` and `cf-admin`.
- Provides built-in retry mechanisms for transient network failures.
- Maintains a durable audit log of all email communications in Supabase.

## Key Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `resend` | ^3.2.0 | Transactional email delivery |
| `drizzle-orm` | ^0.45.2 | Type-safe SQL |
| `postgres` | ^3.4.8 | PostgreSQL driver |
| `@sentry/cloudflare` | ^10.51.0 | Error tracking and distributed tracing |
| `zod` | ^3.25.48 | Queue message validation |

## File Structure
```
cf-email-consumer/
├── src/
│   ├── index.ts              # Queue consumer entry point
│   ├── email/
│   │   ├── dispatcher.ts     # Resend API integration
│   │   └── templates/        # HTML/Text email templates
│   │       ├── booking-admin.ts
│   │       ├── booking-customer.ts
│   │       ├── contact.ts
│   │       └── arco-request.ts
│   ├── db/
│   │   ├── schema.ts         # Drizzle schema definition
│   │   └── audit.ts          # Supabase audit logging logic
│   └── types/                # Queue message interfaces
├── wrangler.toml             # Cloudflare configuration
└── package.json
```
