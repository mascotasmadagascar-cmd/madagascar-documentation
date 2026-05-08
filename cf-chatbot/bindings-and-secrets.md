# cf-chatbot — Bindings & Secrets

> Complete wrangler.jsonc documentation, all secrets with purpose, and environment reference.

---

## Cloudflare Bindings (wrangler.jsonc)

```jsonc
{
  "name": "cf-chatbot-madagascar",
  "main": "src/index.ts",
  "compatibility_date": "2025-03-14",
  "compatibility_flags": ["nodejs_compat"],

  // D1 SQLite database — ISOLATED from cf-admin/cf-astro (different database)
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "chatbot-kb",
      "database_id": "<chatbot-kb-database-id>"  // Different from madagascar-db
    }
  ],

  // Vectorize vector database — semantic search index
  "vectorize": [
    {
      "binding": "VECTOR_INDEX",
      "index_name": "chatbot-kb-index"
    }
  ],

  // Workers AI — embedding generation + last-resort LLM fallback
  "ai": {
    "binding": "AI"
  },

  // Cron trigger — daily conversation cleanup at 2 AM UTC
  "triggers": {
    "crons": ["0 2 * * *"]
  },

  // Custom routes — maps domain to this Worker
  "routes": [
    {
      "pattern": "charlar.madagascarhotelags.com/*",
      "zone_name": "madagascarhotelags.com"
    }
  ]
}
```

**Notable**: cf-chatbot does NOT use:
- R2 (no file storage needed)
- KV (conversation state is in Supabase; response cache is in Upstash)
- Queues (does not produce emails — escalation emails sent directly via Resend)
- Analytics Engine (telemetry written to Supabase instead)

---

## Secrets Registry

Set via `wrangler secret put <KEY>` from the `cf-chatbot/` directory:

### External LLM APIs

| Secret | Purpose | Where to Get |
|--------|---------|-------------|
| `GEMINI_API_KEY` | Google Gemini 2.0 Flash (primary LLM) | Google AI Studio → API Keys |
| `ANTHROPIC_API_KEY` | Claude 3.5 Haiku (fallback LLM) | Anthropic Console → API Keys |

**Cost management**: Both keys should have spending limits configured in their respective consoles. Gemini: $10–20/month max. Anthropic: $5–10/month max (fallback only).

### WhatsApp Integration

| Secret | Purpose |
|--------|---------|
| `WHATSAPP_APP_SECRET` | HMAC-SHA256 signature verification for incoming messages |
| `WHATSAPP_ACCESS_TOKEN` | Meta API token for sending outbound messages |
| `WHATSAPP_PHONE_NUMBER_ID` | The Meta phone number ID for the hotel's WhatsApp number |
| `WHATSAPP_VERIFY_TOKEN` | One-time webhook verification token (set during Meta webhook setup) |

### Database & Caching

| Secret | Purpose |
|--------|---------|
| `SUPABASE_SERVICE_ROLE_KEY` | Full Supabase access (reads/writes conversations, analytics) |
| `SUPABASE_URL` | Supabase project URL (could also be a var, but stored as secret for parity) |
| `UPSTASH_REDIS_REST_URL` | Upstash REST endpoint for rate limiting + response cache |
| `UPSTASH_REDIS_REST_TOKEN` | Upstash authentication token |

### Observability

| Secret | Purpose |
|--------|---------|
| `SENTRY_DSN` | Sentry error reporting endpoint |

### Admin API

| Secret | Purpose | Must Match |
|--------|---------|-----------|
| `ADMIN_API_KEY` | Validates X-Admin-Key header from cf-admin proxy | `CHATBOT_ADMIN_API_KEY` in cf-admin |

### Optional

| Secret | Purpose |
|--------|---------|
| `ESCALATION_EMAIL` | Email address to notify on escalation events |
| `RESEND_API_KEY` | For direct escalation email sending (bypasses queue) |

---

## Local Development (.dev.vars)

```env
# LLM providers
GEMINI_API_KEY=AIza...
ANTHROPIC_API_KEY=sk-ant-...

# WhatsApp (use test credentials for local dev)
WHATSAPP_APP_SECRET=test-secret
WHATSAPP_ACCESS_TOKEN=test-token
WHATSAPP_PHONE_NUMBER_ID=test-phone-id
WHATSAPP_VERIFY_TOKEN=test-verify-token

# Supabase
SUPABASE_URL=https://zlvmrepvypucvbyfbpjj.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Upstash
UPSTASH_REDIS_REST_URL=https://us1-example.upstash.io
UPSTASH_REDIS_REST_TOKEN=Abcd...

# Sentry
SENTRY_DSN=https://...@sentry.io/...

# Admin API (must match CHATBOT_ADMIN_API_KEY in cf-admin .dev.vars)
ADMIN_API_KEY=your-admin-api-key

# Escalation
ESCALATION_EMAIL=harshil.8136@gmail.com
RESEND_API_KEY=re_...
```

---

## Deployment

```bash
cd cf-chatbot

# Set all secrets
wrangler secret put GEMINI_API_KEY
wrangler secret put ANTHROPIC_API_KEY
wrangler secret put WHATSAPP_APP_SECRET
wrangler secret put WHATSAPP_ACCESS_TOKEN
wrangler secret put WHATSAPP_PHONE_NUMBER_ID
wrangler secret put WHATSAPP_VERIFY_TOKEN
wrangler secret put SUPABASE_SERVICE_ROLE_KEY
wrangler secret put SUPABASE_URL
wrangler secret put UPSTASH_REDIS_REST_URL
wrangler secret put UPSTASH_REDIS_REST_TOKEN
wrangler secret put SENTRY_DSN
wrangler secret put ADMIN_API_KEY
wrangler secret put ESCALATION_EMAIL
wrangler secret put RESEND_API_KEY

# Create Vectorize index (one-time setup)
wrangler vectorize create chatbot-kb-index --dimensions=384 --metric=cosine

# Deploy
npm run deploy
# or: wrangler deploy
```

---

## Database IDs Quick Reference

| Resource | ID/Name | Isolated? |
|----------|---------|----------|
| D1 database | `chatbot-kb` (separate from `madagascar-db`) | Yes — chatbot exclusive |
| Vectorize index | `chatbot-kb-index` | Yes — chatbot exclusive |
| Upstash instance | Shared with cf-astro and cf-admin | No — uses `chat:` key prefix |
| Supabase project | `zlvmrepvypucvbyfbpjj` (shared) | No — uses separate tables |
