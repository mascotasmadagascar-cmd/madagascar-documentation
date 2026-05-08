# cf-chatbot — AI Pipeline

> RAG hybrid search, RRF merging, multi-model fallback chain, and response caching.

---

## Pipeline Overview

Every non-trivial customer message goes through a 5-stage pipeline:

```
User message
     │
     ▼
1. INTENT CLASSIFICATION
   Classify into 5 categories
     │
     ├─ Static shortcut → immediate response (no LLM)
     ├─ Escalation → human handoff message
     └─ AI response needed →
          │
          ▼
2. HYBRID KNOWLEDGE BASE SEARCH
   Vectorize (semantic) + D1 FTS5 (keyword) → RRF merge
          │
          ▼
3. CONTEXT ASSEMBLY
   System prompt + KB results + conversation history
          │
          ▼
4. RESPONSE CACHE CHECK (Upstash)
   HIT → return cached response
   MISS →
          │
          ▼
5. LLM GENERATION (multi-model fallback)
   Gemini → Claude → Workers AI → Static
          │
          ▼
   Cache response + return to user
```

---

## Stage 1: Intent Classification

Before any vector search or LLM call, every message is classified to route it optimally:

| Intent | Examples | Action |
|--------|---------|--------|
| `greeting` | "Hola", "Buenos días", "Hello" | Return static greeting (0 LLM cost) |
| `farewell` | "Gracias", "Adiós", "Bye" | Return static farewell (0 LLM cost) |
| `booking_inquiry` | "¿Puedo reservar para el fin de semana?" | Use booking-specialized prompt template |
| `services_question` | "¿Qué razas aceptan?" | Use services prompt template |
| `general` | Most other questions | Use general prompt template |
| `escalation` | "Urgente", "emergencia", "necesito hablar con alguien" | Escalation message + email alert |

Classification is performed by a lightweight pattern-match + small LLM call (or rule-based detection for greetings/farewells to avoid any LLM cost for simple cases).

---

## Stage 2: Hybrid Knowledge Base Search

The KB retrieval uses **two complementary search methods** merged via Reciprocal Rank Fusion:

### 2a. Vectorize — Semantic Search

Finds KB entries that are *conceptually* similar to the query, even if they use different words:

```typescript
// src/core/rag.ts
const embedding = await env.AI.run('@cf/baai/bge-small-en-v1.5', {
  text: normalizedQuery
});

const vectorResults = await env.VECTOR_INDEX.query(embedding.data[0], {
  topK: 10,
  returnMetadata: 'all'
});
```

- **Model**: `bge-small-en-v1.5` (384 dimensions, runs on Workers AI — free)
- **Index**: `chatbot-kb-index` (cosine similarity metric)
- **Returns**: Top-K vector matches with similarity scores

### 2b. D1 FTS5 — Keyword Search

Finds KB entries containing the exact words (or word stems) from the query:

```sql
-- src/storage/d1.ts
SELECT ke.id, ke.content, ke.topic, bm25(kb_entries_fts) AS rank
FROM kb_entries_fts
JOIN kb_entries ke ON ke.id = kb_entries_fts.rowid
WHERE kb_entries_fts MATCH ?
ORDER BY rank
LIMIT 10;
```

- **Engine**: SQLite FTS5 virtual table with BM25 ranking
- **Maintained by**: 3 triggers on `kb_entries` (INSERT/UPDATE/DELETE → sync to FTS5)
- **Advantage**: Exact product names, specific numbers, acronyms — things vectors miss

> ⚠️ **Known Issue**: The current vector ID retrieval uses raw string interpolation:
> ```typescript
> // POTENTIAL SQL INJECTION — IDs from Vectorize are UUIDs, but this pattern is risky
> WHERE id IN (${matchedIds.join(',')})
> ```
> This is in the backlog for remediation using parameterized queries or Drizzle ORM. See [Implementations & Blockers](../implementations-and-blockers.md).

### 2c. Reciprocal Rank Fusion (RRF) Merge

Results from both searches are merged using RRF to produce a single ranked list:

```
RRF score formula: score(d) = Σ  1 / (k + rank_i(d))
                             i∈systems

Where: k = 60 (damping constant)
       rank_i(d) = rank of document d in system i (1-indexed)
```

**Example merge**:
```
Vector results:  doc_A rank 1, doc_B rank 2, doc_C rank 3
FTS5 results:    doc_B rank 1, doc_D rank 2, doc_A rank 3

RRF scores:
  doc_A: 1/(60+1) + 1/(60+3) = 0.01639 + 0.01563 = 0.03202
  doc_B: 1/(60+2) + 1/(60+1) = 0.01613 + 0.01639 = 0.03252 ← wins
  doc_C: 1/(60+3) + 0 = 0.01563
  doc_D: 0 + 1/(60+2) = 0.01613

Final order: doc_B, doc_A, doc_D, doc_C
```

RRF consistently outperforms either search method alone because it rewards documents that appear in multiple result sets.

---

## Stage 3: Context Assembly

The final prompt sent to the LLM is assembled from:

1. **System prompt**: Hotel-specific instructions, persona, escalation rules, language instructions
2. **KB results**: Top-3 to top-5 merged KB entries (trimmed to fit context window)
3. **Conversation history**: Last N messages + running summary (for long conversations)
4. **Intent-specific instructions**: Booking prompt template appended for booking intents

Token budget management ensures the assembled context does not exceed the model's context window.

---

## Stage 4: Upstash Semantic Cache

Before calling any LLM, the system checks for a cached response for the same (or similar) query:

**Cache key formula**:
```typescript
const cacheKey = `chat:${hash(
  intent + 
  kbDocs.map(d => d.id).sort().join(',') + 
  normalizeQuery(userMessage)
)}`;
```

**Cache key components**:
- `intent`: The classified intent (queries with the same intent share cache space)
- `kbDocs.ids`: The retrieved KB document IDs (same KB context = same cache partition)
- `normalizedQuery`: Lowercased, punctuation-stripped, whitespace-normalized query

**Cache TTL**: 24 hours (common questions have stable answers)

**Cache hit rate target**: >40% (many customers ask the same questions)

**Cache miss → Stage 5** (LLM call)
**Cache hit → return immediately** (sub-10ms response, $0 LLM cost)

---

## Stage 5: Multi-Model LLM Fallback Chain

If the cache misses, the system tries LLM providers in priority order:

```
┌─────────────────────────────────────────────────────────────┐
│ Tier 1: Gemini 2.0 Flash (PRIMARY)                          │
│  Provider: Google AI API                                    │
│  Timeout: 8 seconds                                         │
│  Cost: ~$0.001/conversation                                 │
│  Why primary: Fastest, cheapest, excellent multilingual     │
└─────────────────────────┬───────────────────────────────────┘
                          │ FAIL (timeout or error)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ Tier 2: Claude 3.5 Haiku (FALLBACK)                        │
│  Provider: Anthropic API                                    │
│  Timeout: 10 seconds                                        │
│  Cost: ~$0.029/conversation                                 │
│  Why here: Reliable, excellent instruction following        │
└─────────────────────────┬───────────────────────────────────┘
                          │ FAIL (timeout or error)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ Tier 3: Workers AI — Llama (LAST RESORT)                    │
│  Provider: Cloudflare Workers AI (edge inference)           │
│  Timeout: No timeout (synchronous, runs on CF infrastructure)│
│  Cost: Free (up to 10,000 neurons/day)                      │
│  Why last: No internet dependency; always available if CF up │
│  Tradeoff: Smaller model, lower quality, slower             │
└─────────────────────────┬───────────────────────────────────┘
                          │ FAIL (impossible if CF is running)
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ Tier 4: Static Fallback                                     │
│  Hardcoded apologetic message                               │
│  "Lo siento, estoy teniendo dificultades..."               │
└─────────────────────────────────────────────────────────────┘
```

### Timeout Strategy

The timeout chain ensures the chatbot always responds within 20 seconds:
- Gemini: 8s timeout → if exceeded, immediately tries Claude
- Claude: 10s timeout → if exceeded, immediately tries Workers AI
- Workers AI: synchronous, no timeout — if CF is up, it responds
- Total worst case: ~18–20 seconds (only when both external APIs are down)

### Response Caching (Post-LLM)

After a successful LLM response, the result is cached in Upstash with the same key formula, ensuring the LLM is called at most once per unique (intent, KB context, query) combination.
