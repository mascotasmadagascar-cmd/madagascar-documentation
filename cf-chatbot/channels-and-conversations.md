# cf-chatbot — Channels & Conversation Lifecycle

> WhatsApp webhook integration, web widget CORS policy, session management, and conversation cleanup.

---

## Channels

cf-chatbot receives messages from two channels, both routing through the same AI pipeline:

| Channel | Endpoint | Auth Method | Rate Limit |
|---------|---------|-------------|------------|
| **WhatsApp** | `POST /api/webhook/whatsapp` | HMAC-SHA256 signature | Meta platform rate limits |
| **Web Widget** | `POST /api/webhook/web` | CORS origin check | 20 messages/hour/IP (Upstash) |

---

## WhatsApp Channel

### Meta Webhook Setup

The WhatsApp integration uses the Meta Cloud API (Graph API v18+). Meta sends all incoming messages to the registered webhook URL.

**Webhook URL**: `https://charlar.madagascarhotelags.com/api/webhook/whatsapp`

### Verification Handshake (One-Time Setup)

Meta performs a GET request to verify the webhook endpoint during setup:

```
GET /api/webhook/whatsapp
  ?hub.mode=subscribe
  &hub.verify_token={WHATSAPP_VERIFY_TOKEN}
  &hub.challenge={random_string}

cf-chatbot response: {random_string}  ← must echo challenge if token matches
```

### Message Signature Verification (Every Incoming Message)

Meta signs every webhook payload with HMAC-SHA256:

```typescript
// src/api/webhook.ts
const signature = request.headers.get('X-Hub-Signature-256');
// Format: sha256={hex_digest}

const expectedSignature = 'sha256=' + 
  createHmac('sha256', env.WHATSAPP_APP_SECRET)
    .update(rawBody)
    .digest('hex');

if (!timingSafeEqual(signature, expectedSignature)) {
  return new Response('Forbidden', { status: 403 });
}
```

> **Security**: Timing-safe comparison prevents timing oracle attacks. Raw body must be compared (not parsed JSON) — Meta signs the original bytes.

### Message Routing

Incoming WhatsApp message structure:
```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "changes": [{
      "value": {
        "messages": [{
          "from": "+52449...",     // Customer WhatsApp number
          "text": { "body": "¿Tienen disponibilidad para este fin de semana?" },
          "timestamp": "1715..."
        }]
      }
    }]
  }]
}
```

The `from` phone number becomes the `sessionId` for conversation continuity (hashed before storage).

### Replying to WhatsApp

After generating a response, cf-chatbot calls the Meta API:

```typescript
await fetch(`https://graph.facebook.com/v18.0/${env.WHATSAPP_PHONE_NUMBER_ID}/messages`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${env.WHATSAPP_ACCESS_TOKEN}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    messaging_product: 'whatsapp',
    to: customerPhoneNumber,
    text: { body: aiResponse }
  })
});
```

**Message format constraints**: WhatsApp text messages max 4,096 characters. AI responses are truncated if needed, with a note to continue the conversation.

---

## Web Widget Channel

### CORS Policy

The web widget is embedded on `madagascarhotelags.com`. The cf-chatbot allows cross-origin requests only from approved origins:

```typescript
// src/index.ts (Hono middleware)
const ALLOWED_ORIGINS = [
  'https://madagascarhotelags.com',
  'https://www.madagascarhotelags.com',
  'http://localhost:4321',  // cf-astro local dev
];

app.use('/api/webhook/web', cors({
  origin: (origin) => ALLOWED_ORIGINS.includes(origin) ? origin : null
}));
```

### Rate Limiting

Web widget messages are rate-limited via Upstash sliding window:
- **Limit**: 20 messages per IP address per hour
- **Window type**: Sliding (resets per-request, not on the hour)
- **Identifier**: `chat:ratelimit:{hashedIP}`
- **Response on limit**: `{ error: 'rate_limit_exceeded', retryAfter: 3600 }`

### Widget Embedding

The web widget is a `<script>` tag included in cf-astro's `BaseLayout.astro`. It initializes a floating chat button that opens an iframe pointing to the web widget UI served by cf-chatbot.

---

## Conversation Lifecycle

### Session Creation

A conversation "session" is established when a user sends their first message:

```typescript
interface ConversationSession {
  sessionId: string;           // UUID (web) or hashed phone (WhatsApp)
  channel: 'whatsapp' | 'web';
  language: 'es' | 'en';      // Auto-detected from first message
  createdAt: string;           // ISO timestamp
  lastMessageAt: string;       // Updated on every exchange
  messageCount: number;        // Running count
  summaryGenerated: boolean;   // Whether a rolling summary exists
}
```

Sessions are stored in Supabase `bot_conversations` table.

### Message Storage

Every message exchange is stored in Supabase `bot_messages`:

```sql
-- Stored per turn
INSERT INTO bot_messages (session_id, role, content, intent, kb_doc_ids, model_used, latency_ms)
VALUES ($1, 'user', $userMessage, $intent, $kbIds, NULL, NULL);

INSERT INTO bot_messages (session_id, role, content, intent, kb_doc_ids, model_used, latency_ms)
VALUES ($1, 'assistant', $aiResponse, $intent, $kbIds, $modelName, $latencyMs);
```

Stored via `ctx.waitUntil()` — never blocks the user response.

### Context Window Management

For long conversations, full message history would exceed LLM context windows. The system uses a rolling summary strategy:

1. When `messageCount` exceeds 10 turns: generate a summary of the oldest 8 turns
2. Summary replaces the oldest messages in the context assembly
3. Recent 2–3 turns always included verbatim (for immediate context)
4. Summary stored in `bot_conversations.summary` column

**Summary generation**: Triggered synchronously before the next LLM call when `messageCount` is a multiple of 10. Uses the primary LLM (Gemini) for summary generation.

### Escalation

Escalation is triggered when:
- User explicitly asks for human help ("quiero hablar con alguien", "emergency", "urgente")
- User intent is repeatedly unresolved after 3 attempts
- Message contains keywords in the escalation watchlist (`env.ESCALATION_KEYWORDS`)

**Escalation response**:
1. AI sends a message acknowledging the escalation with the hotel's phone number and WhatsApp direct link
2. Optional: cf-chatbot sends a notification email to `env.ESCALATION_EMAIL` (via Resend directly, not the queue)
3. Session is flagged as `escalated = true` in Supabase

---

## Cron: Conversation Cleanup

A daily cron runs at **2:00 AM UTC** (`0 2 * * *`) to maintain database hygiene:

### What Gets Cleaned Up

| Operation | Target | Retention |
|-----------|--------|-----------|
| Delete old messages | `bot_messages` older than 90 days | 90 days |
| Archive old sessions | `bot_conversations` inactive for 30 days | Moved to archive table |
| Prune Upstash cache | Cache entries with expired TTL | Auto-managed by Upstash |
| Regenerate summaries | Sessions with high message count | Refreshes rolling summaries |

### Why 2 AM UTC?

- 2 AM UTC = 8 PM Mexico time (Aguascalientes, CST/CDT) — lowest chatbot traffic period
- Reduces risk of cleanup operations competing with active user sessions

### Cron Configuration

```toml
# cf-chatbot/wrangler.jsonc
[triggers]
crons = ["0 2 * * *"]
```

The cron handler is in `src/index.ts` as the `scheduled` export alongside the `fetch` export.
