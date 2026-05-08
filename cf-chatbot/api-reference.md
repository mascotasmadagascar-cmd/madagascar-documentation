# cf-chatbot — API Reference & Integrations

> External API endpoints (WhatsApp, Web Widget) and Internal API proxy.

---

## 1. Client APIs (External)

These endpoints handle incoming messages from users across different channels.

### `POST /api/webhook/whatsapp`
Handles incoming messages from the WhatsApp Business API.

**Authentication:** 
- Initial webhook verification uses `WHATSAPP_VERIFY_TOKEN` (GET request).
- Incoming message payloads are validated using the `X-Hub-Signature-256` header to ensure they originated from Meta.

**Flow:**
1. Validates Meta signature.
2. Extracts user phone number and message body.
3. Retrieves conversation history from Supabase.
4. Executes `runAIPipeline`.
5. Dispatches response back to user via WhatsApp Graph API (`WHATSAPP_API_TOKEN`).
6. Acknowledges receipt to Meta (Returns 200 OK early using `waitUntil` to prevent Meta retries).

### `POST /api/webhook/web`
Handles messages from the website chat widget.

**Authentication:**
- CORS restricted to `ALLOWED_ORIGINS` (e.g., `https://madagascarhotelags.com`).
- Rate limited via Upstash Redis (e.g., 20 msgs / hr / IP) to prevent LLM abuse.

**Request Body:**
```json
{
  "sessionId": "uuid-v4",
  "message": "Do you accept large dogs?",
  "language": "en"
}
```

**Response (200):**
```json
{
  "response": "Yes, we gladly accept large breed dogs! However, please note that...",
  "sources": ["Large Dog Policy"],
  "model_used": "gemini-1.5-flash"
}
```

---

## 2. Admin APIs (Internal Proxy)

These endpoints are strictly for internal use by `cf-admin` to manage the chatbot's configuration and knowledge base.

**Authentication:** 
All endpoints under `/api/admin/*` require the `X-Admin-Key` header, which must match the `ADMIN_API_KEY` environment variable. The `cf-admin` server proxies these requests, so the key is never exposed to the browser.

### `GET /api/admin/kb`
Retrieves all Knowledge Base entries from D1.

**Response (200):**
```json
{
  "data": [
    {
      "id": 1,
      "title": "Vaccination Requirements",
      "category": "health",
      "content": "All dogs must have...",
      "created_at": "2026-05-01T10:00:00Z",
      "updated_at": "2026-05-01T10:00:00Z"
    }
  ]
}
```

### `POST /api/admin/kb`
Creates a new KB entry.

**Request Body:**
```json
{
  "title": "Holiday Surcharges",
  "category": "pricing",
  "content": "A 15% surcharge applies during..."
}
```

**Side Effects:**
1. Inserts row into D1 `kb_entries`.
2. D1 automatically syncs to `kb_entries_fts` virtual table.
3. Triggers Workers AI to generate embeddings for the content.
4. Inserts new embeddings into Vectorize.

**Response (200):**
```json
{
  "success": true,
  "id": 2,
  "message": "KB entry created and vectorized"
}
```

### `PUT /api/admin/kb/:id`
Updates an existing KB entry and regenerates its embeddings in Vectorize.

### `DELETE /api/admin/kb/:id`
Deletes a KB entry from D1 and removes its corresponding vectors from Vectorize.

---

## 3. System APIs

### `GET /api/health`
Basic health check. Validates connections to D1, Vectorize, and Supabase.

**Response (200):**
```json
{
  "status": "healthy",
  "vectorize": "connected",
  "d1": "connected"
}
```

---

## Lifecycle Cron Triggers

```jsonc
// wrangler.jsonc
"triggers": {
  "crons": ["0 2 * * *"] // Run every day at 2:00 AM
}
```

### Conversation Cleanup Task
The cron trigger executes a maintenance task that:
1. Queries Supabase for conversations inactive for > 24 hours with `status = 'active'`.
2. Updates their status to `closed`.
3. Calculates final conversation metrics (total turns, total cost, primary model).
4. Prunes stale conversation context from Upstash Redis to free memory.
