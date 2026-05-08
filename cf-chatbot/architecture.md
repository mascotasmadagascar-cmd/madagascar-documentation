# cf-chatbot — AI Architecture

> Orchestrator flow, multi-model fallback, and the RAG pipeline.

---

## The AI Orchestration Pipeline

The core logic resides in `runAIPipeline` within `src/core/ai.ts`. This pipeline processes every incoming message through a series of specialized steps to ensure speed, accuracy, and resilience.

```mermaid
graph TD
    A[Incoming User Message] --> B[Intent Classifier]
    B --> C{Cache Check (Upstash)}
    C -->|Hit| D[Return Cached Response]
    C -->|Miss| E[RAG Pipeline]
    
    subgraph "Retrieval Augmented Generation"
        E --> F1[Vectorize: Semantic Search]
        E --> F2[D1 FTS5: Keyword Search]
        F1 --> G[Reciprocal Rank Fusion]
        F2 --> G
        G --> H[Assemble Context Window]
    end

    H --> I[Prompt Assembly based on Intent]
    
    subgraph "Multi-Model Execution Chain"
        I --> J1{Primary: Gemini 1.5 Flash}
        J1 -->|Success| K[Response]
        J1 -->|Timeout/Error| J2{Fallback: Claude 3 Haiku}
        J2 -->|Success| K
        J2 -->|Timeout/Error| J3{Last Resort: Workers AI}
        J3 -->|Success| K
        J3 -->|Fail| L[Canned Error Message]
    end

    K --> M[Cache Store (Upstash)]
    M --> N[Async Analytics Sync]
    N --> O[Return Response to User]
```

---

## 1. Intent Classification

Before querying the KB or generating a response, the system attempts to classify the user's intent. This allows the system to use specialized prompt templates and bypass expensive RAG steps if unnecessary.

| Intent | Action | RAG Used? |
|--------|--------|-----------|
| `greeting` | Return lightweight greeting prompt | No |
| `booking_inquiry` | Use booking-specific prompt, inject live availability | Yes |
| `policy_question` | Use strict policy prompt | Yes |
| `complaint` | Use empathetic prompt, flag for human escalation | Yes |
| `unknown` | Default general inquiry prompt | Yes |

---

## 2. The Hybrid RAG Pipeline

Standard RAG relies purely on semantic search (Vector DBs), which can fail on specific keywords (e.g., "SKU1234"). We use **Hybrid Search** combining dense vectors and sparse text search.

### Step 2a: Vector Search (Semantic)
Uses Cloudflare Vectorize. The user's query is embedded using Workers AI (`@cf/baai/bge-small-en-v1.5`), creating a 384-dimensional vector. Vectorize performs a cosine similarity search to find conceptually related KB entries.

### Step 2b: Full-Text Search (Keyword)
Uses Cloudflare D1's FTS5 extension. The user's query is searched against the `kb_entries_fts` virtual table, finding exact keyword matches.

### Step 2c: Reciprocal Rank Fusion (RRF)
The results from Vectorize and D1 are merged using the RRF algorithm.

```typescript
// RRF Formula
const k = 60;
let score = 0;
if (vectorRank) score += 1 / (k + vectorRank);
if (ftsRank) score += 1 / (k + ftsRank);
```

The top 3-5 documents with the highest combined RRF scores are injected into the LLM context window.

---

## 3. The Multi-Model Fallback Chain

LLM APIs can be unstable or hit rate limits. To guarantee high availability, `cf-chatbot` implements a cascading fallback strategy.

### Tier 1: Primary (Gemini 1.5 Flash)
- **Why**: Extremely fast, large context window (1M tokens), very cost-effective.
- **Role**: Handles 95%+ of all queries.
- **Timeout**: 8 seconds.

### Tier 2: Secondary (Claude 3 Haiku)
- **Why**: Comparable speed and cost to Gemini Flash, excellent instruction following.
- **Role**: Catches queries if Gemini API is down or rate-limited.
- **Timeout**: 10 seconds.

### Tier 3: Edge Fallback (Workers AI)
- **Why**: Runs directly on Cloudflare's edge network, no external API dependency.
- **Model**: `@cf/meta/llama-3-8b-instruct`.
- **Role**: Absolute last resort if external internet routing fails or both primary APIs are down. Ensures the bot never completely fails.

All models adhere to a standard interface, streaming normalized chunks back to the orchestrator.

---

## 4. Semantic Caching (Upstash Redis)

To reduce latency and LLM costs, responses are cached in Upstash Redis.

**Cache Key Generation:**
The cache key is a hash of the user's intent, the retrieved KB document IDs, and the normalized query text.

```typescript
const cacheKey = `chat:${hash(intent + kbDocs.map(d => d.id).join(',') + normalizedQuery)}`;
```

If a user asks "What time is check-in?" and another asks "When can I drop off my dog?", if they map to the same intent and retrieve the exact same KB documents, the second query is served from Redis in ~10ms (saving LLM cost and 1s+ of generation time).

---

## 5. Async Analytics & Health Reporting

After a response is generated, metadata is sent to Supabase for the analytics dashboard (`cf-admin`).

**Metrics Captured:**
- Token usage (input/output)
- Model used (Gemini vs. Claude vs. LLaMA)
- Response time (ms)
- KB Hit/Miss (did the RAG pipeline find relevant docs?)
- Estimated cost (calculated based on token usage)

This operation is wrapped in a `waitUntil` block, meaning it executes asynchronously after the response has been sent to the user, ensuring zero latency impact on the chat experience.
