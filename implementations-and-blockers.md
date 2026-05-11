# Implementations & Blockers

> Tracks open operational risks, known gaps, and their remediation steps. Closed items are archived at the bottom.

---

## Open Blockers

### Blocker 1 — Sentry Missing in `cf-chatbot` (HIGH)

**Risk**: Silent failures in the LLM fallback chain, uncaptured Vectorize/D1 errors, zero distributed tracing from the cf-admin proxy boundary.

**Remediation**:

1. Install `@sentry/cloudflare` in `cf-chatbot/`.
2. Wrap the Hono router with `Sentry.wrapFetch` in `src/index.ts`:

```typescript
import * as Sentry from '@sentry/cloudflare';

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext) {
    return Sentry.wrapFetch(
      { dsn: env.SENTRY_DSN, tracesSampleRate: 0.1 },
      request, env, ctx,
      () => app.fetch(request, env, ctx)
    );
  }
};
```

3. Extract `sentry-trace` and `baggage` headers from incoming cf-admin proxy requests to stitch the distributed trace across the service boundary.
4. Add `SENTRY_DSN` secret to cf-chatbot via `wrangler secret put SENTRY_DSN`.
5. Add Sentry Vite plugin to `wrangler.jsonc` for source map upload.

**Affected service**: `cf-chatbot`

---

### Blocker 2 — CJS Module Format in `cf-email-consumer` (MEDIUM)

**Risk**: Larger bundle sizes, potential dependency conflicts with ESM-only packages (Drizzle, Sentry), suboptimal startup time. Future Cloudflare runtime changes may break CJS consumers.

**Remediation**:

1. Add `"type": "module"` to `cf-email-consumer/package.json`.
2. Update `tsconfig.json`:
   ```json
   {
     "compilerOptions": {
       "module": "ESNext",
       "moduleResolution": "bundler",
       "target": "ESNext"
     }
   }
   ```
3. Replace any `require()` calls with `import`.
4. Verify Wrangler build output is ESM (`format = "esm"` in `wrangler.toml` if explicitly set).

**Affected service**: `cf-email-consumer`

---

### Blocker 3 — No Dead Letter Queue Monitoring (HIGHEST PRIORITY)

**Risk**: When a queue message exhausts all 3 retries (e.g., Resend outage lasting >15 minutes), the message is **silently dropped**. Booking confirmation emails are permanently lost with no alert fired and no recovery path.

**Current state**: `dead_letter_queue` is commented out in `cf-email-consumer/wrangler.toml`.

**Remediation steps**:

1. **Create the DLQ queue** in Cloudflare Dashboard → Workers & Pages → Queues → Create Queue:
   - Name: `madagascar-emails-dlq`

2. **Uncomment in `cf-email-consumer/wrangler.toml`**:
   ```toml
   [[queues.consumers]]
   queue = "madagascar-emails"
   max_batch_size = 10
   max_batch_timeout = 5
   max_retries = 3
   dead_letter_queue = "madagascar-emails-dlq"   # ← uncomment this line
   ```

3. **Create a DLQ consumer handler** in `cf-email-consumer/src/dlq-handler.ts`:
   ```typescript
   import * as Sentry from '@sentry/cloudflare';

   export async function handleDLQ(batch: MessageBatch<EmailQueueMessage>, env: Env) {
     for (const msg of batch.messages) {
       Sentry.captureEvent({
         level: 'fatal',
         message: `[DLQ] Email permanently failed after all retries`,
         extra: {
           trackingId: msg.body.trackingId,
           type: msg.body.type,
           projectSource: msg.body.projectSource,
           retryCount: msg.attempts,
         }
       });
       msg.ack();
     }
   }
   ```

4. **Register DLQ consumer** in `wrangler.toml`:
   ```toml
   [[queues.consumers]]
   queue = "madagascar-emails-dlq"
   max_batch_size = 5
   max_batch_timeout = 5
   max_retries = 1
   ```

5. **Redeploy** `cf-email-consumer`.

6. **Verify**: Submit a test booking, confirm DLQ is bound in CF Dashboard → Queue → DLQ tab.

**Affected service**: `cf-email-consumer`

---

## Closed Blockers

### ~~Blocker 4 — HTTP Proxies Instead of Service Bindings~~ ✅ RESOLVED

**Was**: cf-admin communicated with cf-chatbot and cf-astro over public HTTP endpoints, introducing TLS overhead, DNS resolution time, and unnecessary network cost.

**Resolution**: cf-admin `wrangler.toml` now declares both service bindings:

```toml
[[services]]
binding = "CHATBOT_SERVICE"
service = "cf-chatbot"

[[services]]
binding = "ASTRO_SERVICE"
service = "cf-astro"
```

- `/api/chatbot/[...path]` now proxies via `env.CHATBOT_SERVICE.fetch()` (zero-latency internal call)
- CMS revalidation calls `env.ASTRO_SERVICE.fetch()` via `revalidateAstro()` in `src/lib/cms.ts`
- HTTP fallback (`CHATBOT_WORKER_URL`) retained as emergency override only
- No auth header required for service binding calls (binding is implicit trust)

---

### ~~Turnstile Build-Time Import Fix~~ ✅ RESOLVED

**Was**: `TURNSTILE_SITE_KEY` resolved using `import.meta.env` (build-time), causing CF runtime warnings on Contact and ARCO pages.

**Resolution**: Migrated to `getEnv(Astro)` runtime binding in both the widget injection and `src/pages/api/contact.ts` validation. ARCO routes (`/es/legal/arco.astro`, `/en/legal/arco.astro`) migrated from `getStaticPaths` to `export const prerender = false` for SSR.

---

## Monitoring Gaps (Not Blockers, but Worth Tracking)

| Gap | Risk | Suggested Fix |
|-----|------|--------------|
| Analytics Engine data never queried | Business intelligence collected but never used | Build CF API queries or dashboard; see `08-observability-and-monitoring.md` |
| No uptime monitoring | Service outages not detected proactively | Add Cloudflare Health Checks or BetterStack uptime monitor |
| Sentry `head_sampling_rate = 1` (100%) | Bot crawls exhaust Sentry quota | Lower to `0.05` in `cf-astro/wrangler.toml` |
| Supabase free tier auto-pause | Project pauses after 7 days inactivity | Schedule keep-alive ping or upgrade to Pro ($25/month) |
| cf-chatbot has no Sentry | See Blocker 1 | — |
