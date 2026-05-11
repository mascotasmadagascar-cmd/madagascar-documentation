# 12 — Service Connections & Dependency Map

> How all 4 Madagascar services talk to each other, what they share, and what breaks if one goes down.

---

## Service Dependency Overview

```
                          ┌──────────────────────────────────┐
                          │        EXTERNAL WORLD            │
                          │  Customers / Admins / WhatsApp   │
                          └────┬───────────┬─────────────────┘
                               │           │
                    bookings   │           │ admin login
                    contact    │           │ CMS edits
                    chatbot    │           │ user management
                               ▼           ▼
              ┌────────────────────┐  ┌────────────────────────┐
              │     cf-astro       │  │       cf-admin         │
              │  Public Website    │  │   Admin Dashboard      │
              │  madagascarhotel.. │  │  secure.madagascarh..  │
              └─────────┬──────────┘  └────────────┬───────────┘
                        │                           │
              ┌─────────▼──────────────────────────▼──────────┐
              │            madagascar-emails (Queue)           │
              │         Cloudflare Queue — async delivery      │
              └─────────────────────┬──────────────────────────┘
                                    │
                          ┌─────────▼──────────────┐
                          │   cf-email-consumer    │
                          │   Email Queue Worker   │
                          └────────────────────────┘

              ┌──────────────────────────────────────────────┐
              │              cf-chatbot                      │
              │         AI Chatbot (WhatsApp + Web)          │
              │  cf-admin manages it; cf-astro hosts widget  │
              └──────────────────────────────────────────────┘
```

---

## Shared Infrastructure

These resources are physically shared between multiple services. Misconfiguring one affects all users of that resource.

| Resource | Cloudflare ID | Used By | Access Pattern |
|----------|--------------|---------|----------------|
| **D1: `madagascar-db`** | `7fca2a07-d7b4-449d-b446-408f9187d3ca` | cf-astro (read/write), cf-admin (read/write) | D1 binding; NOT shared with chatbot or email-consumer |
| **D1: `chatbot-kb`** | Separate database | cf-chatbot only | D1 binding; isolated from other services |
| **R2: `madagascar-images`** | — | cf-astro (read), cf-admin (read/write) | IMAGES binding; admin uploads, astro serves |
| **R2: `arco-documents`** | — | cf-astro only | ARCO_DOCS binding; private, presigned URLs only |
| **Queue: `madagascar-emails`** | — | cf-astro (produce), cf-admin (produce), cf-email-consumer (consume) | cf-astro and cf-admin push; consumer pops |
| **Analytics Engine: `madagascar_analytics`** | — | cf-astro (write), cf-admin (write) | Both write telemetry; both must use identical dataset name |
| **Supabase PostgreSQL** | `zlvmrepvypucvbyfbpjj` | All 4 services (different roles) | See role matrix below |
| **Upstash Redis** | Single instance | cf-astro (rate limiting), cf-admin (rate limiting), cf-chatbot (response cache) | Shared namespace — key prefixes prevent collision |

### Supabase Connection Role Matrix

| Service | Role | Permission Level | Why |
|---------|------|-----------------|-----|
| cf-astro | `cf_astro_writer` | INSERT on PII tables; SELECT+UPDATE on audit log | Least privilege — cannot read back what it wrote |
| cf-admin | `service_role` | Full access, bypasses RLS | Admin operations require full visibility |
| cf-chatbot | `service_role` | Full access, bypasses RLS | Reads/writes conversations and analytics |
| cf-email-consumer | `cf_astro_writer` (via DATABASE_URL) | SELECT+UPDATE on `email_audit_log` | Only needs to update delivery status |

> ⚠️ **Critical**: `cf-astro` and `cf-email-consumer` share the same `DATABASE_URL` secret and connect as `cf_astro_writer`. If that role's permissions are changed, both services are affected simultaneously.

---

## Inter-Service Communication

### cf-admin → cf-astro (ISR Revalidation)

**Purpose**: When admin edits website content, the public site needs to refresh its cached pages.

**Implementation**: `ASTRO_SERVICE` service binding (zero-latency internal call, no TLS/DNS overhead).

```
cf-admin (content edit)
    │
    ├─ UPDATE D1 cms_content table
    ├─ PUT fresh CMS JSON into cf-astro KV (direct KV injection — bypasses D1 lag)
    │
    └─ env.ASTRO_SERVICE.fetch('/api/revalidate', {
           method: 'POST',
           headers: { Authorization: 'Bearer {REVALIDATION_SECRET}' },
           body: JSON.stringify({ paths: ['/es/', '/en/'], cmsData: { ... } })
       })
           │
           cf-astro verifies REVALIDATION_SECRET
           cf-astro deletes matching KV ISR_CACHE keys (deploy-scoped)
           Next visitor gets fresh SSR → saved back to KV
```

**Retry policy**: 3 attempts with exponential backoff (300ms → 600ms → 900ms). If all fail, D1 content is updated but cached HTML is stale until next visitor triggers fresh SSR or 24h KV TTL expires.

**Shared secret**: `REVALIDATION_SECRET` — must be identical in both services.

---

### cf-admin → cf-chatbot (Admin API Proxy)

**Purpose**: cf-admin hosts the chatbot management UI (KB editor, model config, analytics). All chatbot admin operations proxy through cf-admin to cf-chatbot.

**Implementation**: `CHATBOT_SERVICE` service binding (zero-latency internal call). HTTP fallback via `CHATBOT_WORKER_URL` retained for emergency use only.

```
Admin user in browser
    │
    └─ cf-admin /api/chatbot/[...path]
           │
           ├─ Validates Zero Trust session (role: admin+)
           │
           └─ env.CHATBOT_SERVICE.fetch(
                new Request(`http://internal/api/admin/${path}`, {
                  method, headers, body  // forwarded from admin request
                })
              )
                   │
                   cf-chatbot performs KB CRUD / config change
                   Returns response → cf-admin → browser
```

**No auth header needed**: Service binding is implicit trust — `CHATBOT_ADMIN_API_KEY` is only used when the HTTP fallback (`CHATBOT_WORKER_URL`) is invoked.

### cf-astro / cf-admin → cf-email-consumer (Queue)

**Purpose**: Decouples email sending from the user request. Neither cf-astro nor cf-admin ever call Resend directly.

```
cf-astro (booking submitted)              cf-admin (contact reply)
    │                                          │
    ├─ INSERT email_audit_log (status: 'queued')
    │
    └─ env.EMAIL_QUEUE.send({               env.EMAIL_QUEUE.send({
         type: 'booking_admin',               type: 'contact',
         trackingId: uuid,                    trackingId: uuid,
         payload: { ... },                    payload: { ... }
         sentryTrace: '...',              })
         sentryBaggage: '...'
       })

                         ↓ (async — typically seconds later)

                 cf-email-consumer receives batch
                    │
                    ├─ Validates message with Zod
                    ├─ Generates HTML template
                    ├─ POST Resend API → email delivered
                    ├─ UPDATE email_audit_log (status: 'sent', resend_message_id)
                    └─ msg.ack()  ← removes from queue
```

**Sentry trace stitching**: cf-astro attaches `sentryTrace` and `sentryBaggage` to booking queue messages. cf-email-consumer calls `Sentry.continueTrace()` to create a unified flamegraph spanning the queue boundary.

---

## Request Lifecycle Walkthroughs

### 1. Customer Books a Stay

```
1. Customer fills booking form on madagascarhotelags.com/es/booking
2. Browser submits POST /api/booking
3. cf-astro middleware: check rate limit (Upstash, 5/hr/IP)
4. cf-astro: verify Cloudflare Turnstile token (anti-bot)
5. cf-astro: validate body with Zod BookingSchema
6. cf-astro: INSERT into Supabase (bookings + pets + booking_consents + email_audit_log)
7. cf-astro: env.EMAIL_QUEUE.send(booking_admin message)
8. cf-astro: env.EMAIL_QUEUE.send(booking_customer message)
9. cf-astro: return 200 {bookingRef} to browser ← user sees confirmation in ~200ms
10. [async] cf-email-consumer: send admin notification email via Resend
11. [async] cf-email-consumer: send customer confirmation email via Resend
12. [async] cf-email-consumer: UPDATE email_audit_log (sent)
13. [async] Resend webhook → cf-astro /api/webhooks/resend → UPDATE email_audit_log (delivered)
```

**Services involved**: cf-astro, Supabase, Upstash, Cloudflare Turnstile, Queue, cf-email-consumer, Resend
**User wait**: ~200ms (steps 1–9 only; email sending is background)

### 2. Admin Logs Into the Portal

```
1. Admin navigates to secure.madagascarhotelags.com
2. Cloudflare Zero Trust intercepts: checks for valid CF Access JWT cookie
3. If no JWT: redirect to identity provider (Google or GitHub), authenticate, get JWT
4. CF Access forwards request to cf-admin Worker with CF-Access-JWT + CF-Access-Email headers
5. cf-admin middleware: verify JWT (RS256, AUD check, JWK from CF issuer endpoint)
6. cf-admin: query Supabase admin_authorized_users by email → verify active + get role
7. cf-admin: compute PLAC access map from D1 admin_pages table
8. cf-admin: CREATE KV session (24h TTL): {userId, email, role, accessMap, ...}
9. cf-admin: SET session cookie
10. [waitUntil] cf-admin: INSERT login_forensics row into D1
11. [waitUntil] cf-admin: UPSERT cf_sub_id into Supabase
12. Admin sees dashboard ← full auth in ~300ms
```

**Subsequent requests**: cf-admin reads KV session (step 8 already done) → ~5ms overhead per request. Role re-checked from Supabase every 30 minutes.

### 3. AI Chatbot Responds to WhatsApp Message

```
1. Customer sends WhatsApp message to hotel's number
2. Meta forwards to: POST https://charlar.madagascarhotelags.com/api/webhook/whatsapp
3. cf-chatbot: verify HMAC-SHA256 signature (WHATSAPP_VERIFY_TOKEN)
4. cf-chatbot: classify intent (booking inquiry / services / general / escalation / greeting)
5. If greeting/farewell: return static response (no LLM cost)
6. If escalation needed: return escalation message + send email alert
7. Otherwise:
   a. Query Vectorize: top-K semantic matches from KB
   b. Query D1 FTS5: top-K keyword matches from KB
   c. Merge results via RRF (Reciprocal Rank Fusion, k=60)
   d. Load conversation history from Supabase (last N messages + summary)
   e. Check Upstash cache: key = hash(intent + kbDocIds + normalizedQuery)
   f. Cache HIT → return cached response immediately
   g. Cache MISS → call Gemini 2.0 Flash (8s timeout)
      → if fail: call Claude 3.5 Haiku (10s timeout)
      → if fail: call Workers AI Llama
      → cache result in Upstash
8. [waitUntil] Save message + response to Supabase conversations table
9. [waitUntil] Write analytics event to Supabase
10. Return response to WhatsApp via Meta API
```

**Services involved**: cf-chatbot, Vectorize, D1, Supabase, Upstash, Gemini API, Anthropic API (fallback), Workers AI (last resort)

### 4. Admin Updates Website Content (CMS)

```
1. Admin edits hero text in cf-admin Content Studio
2. Browser submits PATCH to cf-admin /api/content/blocks
3. cf-admin: verify admin session + RBAC (editor or above)
4. cf-admin: UPDATE D1 cms_content table
5. cf-admin: PUT new content into KV ISR_CACHE (direct injection — bypasses D1 lag)
6. cf-admin: env.ASTRO_SERVICE.fetch('/api/revalidate') [Service Binding — zero-latency]
7. cf-astro: verify REVALIDATION_SECRET
8. cf-astro: DELETE KV keys matching isr:{buildId}:/es/* and isr:{buildId}:/en/*
9. Admin sees success toast ← ~400ms total
10. Next visitor to the homepage: cf-astro misses KV → renders fresh SSR → saves to KV
```

**Services involved**: cf-admin, D1, KV, cf-astro (HTTP webhook)
**Result**: Content is live on the public website within seconds of the admin saving

---

## Shared Secrets Reference

Secrets that must be kept in sync across services — if they drift, cross-service calls break silently:

| Secret | Value In | Also Required In | Consequence of Mismatch |
|--------|----------|-----------------|------------------------|
| `REVALIDATION_SECRET` | cf-admin | cf-astro | ISR revalidation silently fails; stale content served |
| `CHATBOT_ADMIN_API_KEY` | cf-admin (as `CHATBOT_ADMIN_API_KEY`) | cf-chatbot (as `ADMIN_API_KEY`) | All chatbot admin operations return 401 |
| `DATABASE_URL` | cf-astro (cf_astro_writer) | cf-email-consumer (same role) | Email audit log updates fail; delivery status unknown |
| `SUPABASE_SERVICE_ROLE_KEY` | cf-admin | cf-chatbot | Supabase operations fail entirely for that service |

> **When rotating a shared secret**: update ALL services simultaneously, then redeploy in this order: cf-chatbot → cf-email-consumer → cf-astro → cf-admin (lowest to highest user impact).

---

## What Breaks If Each Service Goes Down

| Service Down | Immediate Impact | Degraded Behavior | Data Loss Risk |
|-------------|-----------------|------------------|----------------|
| **cf-astro** | Public website unreachable | Booking forms inaccessible | None (Cloudflare edges serve error page) |
| **cf-admin** | Admin portal unreachable | Staff cannot manage bookings or content | None |
| **cf-chatbot** | Chatbot offline | WhatsApp messages unanswered; web widget error | In-flight messages may be lost |
| **cf-email-consumer** | Email delivery paused | Queue holds messages (24h free tier retention) | Messages older than 24h on free plan |
| **Supabase** | Bookings fail | cf-astro booking API returns 503; cf-admin shows stale data | No data loss (transactions fail cleanly) |
| **Upstash** | Rate limiting disabled | Forms accept unlimited submissions (spam risk); chatbot cache misses (higher LLM cost) | None |
| **Resend** | Email delivery paused | Queue holds messages; audit log shows 'queued' status | None (queue retries for up to 24h) |
| **Gemini API** | Chatbot auto-falls to Claude | Slightly higher latency + cost; Workers AI is last resort | None |
| **Cloudflare edge** | Everything unreachable | — | None (edge failures are extremely rare) |

---

## Cloudflare Account Requirements

> 🚨 **Critical**: Both cf-astro and cf-admin MUST be deployed to the **same Cloudflare account** to share D1 and R2 bindings. Using different accounts causes 404/auth errors at runtime.

| Requirement | Value |
|-------------|-------|
| Account ID | `320d1ebab5143958d2acd481ea465f52` (Mascotas Madagascar) |
| All 4 services deployed to | Same account |
| D1 database ID must match | `7fca2a07-d7b4-449d-b446-408f9187d3ca` in both cf-astro and cf-admin wrangler.toml |
| R2 bucket name must match | `madagascar-images` in both cf-astro and cf-admin |
