# cf-chatbot — Analytics & Costs

> What's tracked, where it goes, LLM cost per conversation, and free tier utilization.

---

## What's Tracked

### Per-Conversation Metrics (Supabase)

Every conversation session records:

| Metric | Column | Description |
|--------|--------|-------------|
| Session ID | `bot_conversations.id` | Anonymous UUID |
| Channel | `bot_conversations.channel` | `whatsapp` or `web` |
| Language | `bot_conversations.language` | `es` or `en` |
| Message count | `bot_conversations.message_count` | Total turns |
| Start/end time | `created_at`, `last_message_at` | Duration derivable |
| Escalation | `bot_conversations.escalated` | Whether human handoff requested |
| Resolution | `bot_conversations.resolved` | Admin-set after review |

### Per-Message Metrics (Supabase)

```sql
bot_messages (
  session_id,
  role,           -- 'user' | 'assistant'
  content,        -- Full text (for review)
  intent,         -- Classified intent
  kb_doc_ids,     -- JSON array of KB entry IDs that contributed to response
  model_used,     -- 'gemini-2.0-flash' | 'claude-3-5-haiku' | 'workers-ai'
  latency_ms,     -- Response generation time
  cache_hit       -- Whether response came from Upstash cache
)
```

### Aggregate Analytics (Cloudflare Analytics Engine)

Real-time operational metrics written per message via `ctx.waitUntil()`:

| Event | Blobs | Doubles |
|-------|-------|---------|
| `chatbot_message` | intent, model, channel | latency_ms, is_cache_hit (0/1), is_escalation (0/1) |
| `chatbot_session_start` | channel, language | — |
| `chatbot_session_end` | channel | message_count, duration_seconds |

These appear in cf-admin's Chatbot Analytics dashboard as:
- Intent distribution chart (booking inquiries vs services vs general)
- Cache hit rate over time
- Model tier usage (what % of responses used Gemini vs Claude vs Workers AI)
- Latency distribution
- Escalation rate

---

## cf-admin Chatbot Dashboard

The cf-admin chatbot section (`/dashboard/chatbot/`) provides:

| Page | Description |
|------|-------------|
| **Analytics** | Real-time message telemetry, intent distribution, response time charts |
| **Conversations** | Browse and search all conversations; review escalated sessions |
| **Knowledge Base** | Add/edit/delete KB entries; trigger reindex |
| **Models** | View model catalog, switch active primary model |
| **Prompts** | Edit system prompt, update escalation keywords |
| **Config** | Set cache TTL, max context messages, other operational settings |
| **Usage** | Token/neuron consumption by model, projected monthly cost |

---

## LLM Cost Per Conversation

### Cost Model

LLM costs are calculated on tokens (roughly 1 token = 0.75 words in English/Spanish).

**Typical conversation** (5 turns, ~2,000 input tokens + 1,500 output tokens total):

| Model | Input Cost per M | Output Cost per M | 5-Turn Conversation Cost |
|-------|-----------------|------------------|--------------------------|
| Gemini 2.0 Flash | ~$0.10 | ~$0.40 | **~$0.001** |
| Claude 3.5 Haiku | ~$0.80 | ~$4.00 | **~$0.008** |
| Workers AI (Llama) | Free | Free | **$0.000** |

### Weighted Average Cost (Current Mix)

Based on observed model fallback rates:
- **95% of conversations**: Gemini 2.0 Flash primary → $0.001/conv
- **4% of conversations**: Claude 3.5 Haiku fallback → $0.008/conv
- **1% of conversations**: Workers AI fallback → $0.000/conv

**Weighted average**: ~$0.0013/conversation

### Volume Projections

| Monthly Conversations | Gemini Cost | Claude Cost | Total LLM |
|----------------------|-------------|-------------|-----------|
| 100 | $0.10 | $0.03 | **~$0.13** |
| 500 | $0.48 | $0.16 | **~$0.64** |
| 1,000 | $0.95 | $0.32 | **~$1.27** |
| 5,000 | $4.75 | $1.60 | **~$6.35** |
| 10,000 | $9.50 | $3.20 | **~$12.70** |

### Cache Impact on Cost

The Upstash semantic cache avoids LLM calls for repeated questions. At target cache hit rate of 40%:
- Effective cost is reduced by 40%
- At 1,000 conversations/month: ~$0.76 instead of $1.27
- Cache is especially effective for: service prices, hours, location, pet requirements (very frequently asked)

---

## Free Tier Utilization

### Cloudflare Workers AI (Embedding Generation)

Used during KB admin operations (adding/updating KB entries):

| Operation | Neurons Used | Frequency |
|-----------|-------------|-----------|
| Generate embedding for new KB entry | ~50–100 | Per KB create/update |
| Llama fallback response | ~1,000–3,000 per turn | Rare (fallback only) |

**Free limit**: 10,000 neurons/day
**Normal usage**: ~50–500 neurons/day (KB admin + occasional fallback)
**Headroom**: ~95–99%

**Risk scenario**: If both Gemini and Claude are down simultaneously, Workers AI handles all conversations. At 50 concurrent fallback conversations, the 10K daily budget exhausts in ~3 hours. This is an edge case — simultaneous failure of both external APIs is extremely rare.

### Vectorize (Knowledge Base Search)

| Metric | Per Message | Monthly at 1K msgs |
|--------|------------|-------------------|
| Dimensions queried | 384 (one query) | 384,000 |
| **Free limit** | 30,000,000/month | — |
| **Headroom** | — | 98.7% |

Vectorize is not a practical constraint at any realistic chatbot volume.

### Upstash Redis (Rate Limiting + Cache)

| Operation | Commands | Daily at 500 msgs |
|-----------|---------|------------------|
| Rate limit check (per message) | 2 | 1,000 |
| Cache GET (per message) | 1 | 500 |
| Cache SET (on miss) | 1 | ~300 (60% miss rate) |
| **Total** | ~4/message | ~1,800/day |
| **Free limit** | 10,000 commands/day | — |
| **Headroom** | — | ~82% |

At 2,000 messages/day, Upstash approaches the 10,000 command limit. Beyond that, upgrade to Pay-as-you-go (~$0.50–1.50/month).

### Supabase Database

| Table | Writes per message | Monthly at 1K msgs |
|-------|-------------------|-------------------|
| `bot_messages` | 2 (user + assistant) | 2,000 rows |
| `bot_conversations` | 0.2 (1 per session, avg 5 msgs/session) | 200 rows |
| **Supabase free limit** | 500 MB DB storage | — |

Chatbot data is negligible relative to Supabase storage limits. 1,000 conversations/month with full message content is ~5–10 MB.

---

## Knowledge Base Gap Analysis

One of the most valuable analytics features: identifying questions the bot couldn't answer well.

**Gap signals**:
- `intent = 'general'` + `escalated = true` in same session → bot didn't have relevant KB content
- Repeated similar questions with low `kb_doc_ids` match count → KB needs entries on that topic
- High `model_used = 'workers-ai'` rate for a topic → that topic returns poor KB results, causing more LLM retries

**Monthly KB review** (recommended): Export sessions where `escalated = true` or `cache_hit = false AND model_used = 'workers-ai'`. Common topics appearing there are candidates for new KB entries.

---

## Monitoring Recommendations

| Metric | Alert Threshold | Action |
|--------|----------------|--------|
| Cache hit rate | < 30% | Review query normalization; add more KB entries for common questions |
| Claude fallback rate | > 10% | Check Gemini API status; investigate timeouts |
| Workers AI fallback rate | > 2% | Check both Gemini and Claude; may indicate API outage |
| Escalation rate | > 15% | KB has gaps; add more content on escalated topics |
| Upstash commands/day | > 8,000 | Upgrade to Pay-as-you-go before hitting 10K limit |
| Avg response latency | > 3,000ms | Check Gemini API latency; consider switching primary model |
