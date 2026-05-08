# cf-chatbot — Knowledge Base

> D1 schema, FTS5 indexing, Vectorize embeddings, admin CRUD flow, and consistency strategy.

---

## Overview

The knowledge base (KB) is the chatbot's "memory" of hotel-specific information. It is stored in two complementary systems that are kept in sync:

| System | Purpose | Search Type |
|--------|---------|-------------|
| D1 `chatbot-kb` database | Source of truth, FTS5 keyword search | Exact keyword matching, BM25 ranking |
| Cloudflare Vectorize | Semantic search index | Conceptual similarity, cosine distance |

Both systems must contain the same KB entries. The D1 database is authoritative; Vectorize is a derived index.

---

## D1 Database Schema

### `kb_entries` — Primary Storage

```sql
CREATE TABLE kb_entries (
  id TEXT PRIMARY KEY DEFAULT (lower(hex(randomblob(16)))),
  topic TEXT NOT NULL,          -- 'pricing', 'services', 'policies', 'location', etc.
  title TEXT NOT NULL,          -- Short descriptive title
  content TEXT NOT NULL,        -- Full text content (what the bot reads)
  language TEXT DEFAULT 'es',   -- 'es' | 'en' | 'both'
  is_active INTEGER DEFAULT 1,  -- 0 = disabled (hidden from search)
  vector_id TEXT,               -- Vectorize vector ID (for sync tracking)
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now'))
);
```

**Topics used**:
- `pricing` — Room rates, service costs, packages
- `services` — List of services offered (grooming, daycare, boarding)
- `policies` — Check-in/check-out times, cancellation policy, deposit requirements
- `pet_requirements` — Accepted species, breeds, vaccination requirements
- `location` — Address, directions, parking, nearby landmarks
- `contact` — Phone, WhatsApp, email, hours of operation
- `faq` — Frequently asked questions

### `kb_entries_fts` — Full-Text Search Index (FTS5 Virtual Table)

```sql
CREATE VIRTUAL TABLE kb_entries_fts USING fts5(
  title,
  content,
  topic UNINDEXED,
  content='kb_entries',
  content_rowid='rowid',
  tokenize='unicode61 remove_diacritics 2'  -- Handles Spanish accents
);
```

The `remove_diacritics 2` tokenizer setting ensures `"habitación"` matches `"habitacion"` — critical for Spanish text search.

### Sync Triggers (3 Triggers)

FTS5 requires manual synchronization with the source table. Three SQLite triggers maintain this automatically:

```sql
-- On INSERT: add to FTS index
CREATE TRIGGER kb_entries_ai AFTER INSERT ON kb_entries BEGIN
  INSERT INTO kb_entries_fts(rowid, title, content, topic)
    VALUES (new.rowid, new.title, new.content, new.topic);
END;

-- On UPDATE: re-index the modified entry
CREATE TRIGGER kb_entries_au AFTER UPDATE ON kb_entries BEGIN
  INSERT INTO kb_entries_fts(kb_entries_fts, rowid, title, content, topic)
    VALUES ('delete', old.rowid, old.title, old.content, old.topic);
  INSERT INTO kb_entries_fts(rowid, title, content, topic)
    VALUES (new.rowid, new.title, new.content, new.topic);
END;

-- On DELETE: remove from FTS index
CREATE TRIGGER kb_entries_ad AFTER DELETE ON kb_entries BEGIN
  INSERT INTO kb_entries_fts(kb_entries_fts, rowid, title, content, topic)
    VALUES ('delete', old.rowid, old.title, old.content, old.topic);
END;
```

### `bot_config` — Chatbot Configuration

```sql
CREATE TABLE bot_config (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  updated_at TEXT DEFAULT (datetime('now'))
);
```

**Config keys**:
- `system_prompt` — The main system prompt (persona, rules, hotel name)
- `escalation_keywords` — JSON array of words that trigger escalation
- `active_model` — Currently selected primary LLM
- `cache_ttl_seconds` — Upstash cache TTL (default: 86400)
- `max_context_messages` — Messages to include before summarizing (default: 10)

---

## Vectorize Index

| Property | Value |
|----------|-------|
| Index name | `chatbot-kb-index` |
| Dimensions | 384 |
| Distance metric | Cosine similarity |
| Embedding model | `@cf/baai/bge-small-en-v1.5` (Workers AI) |
| Max vectors | 10,000,000 (free tier supports up to 200,000 stored) |

**Current usage**: KB typically contains 50–500 entries → well within free tier.

---

## Admin CRUD Flow

KB entries are managed through the cf-admin Knowledge Base editor (`/dashboard/chatbot/kb`). All operations proxy through cf-admin to cf-chatbot's admin API.

### Create Entry

```
Admin types new KB entry in cf-admin UI
    │
    ├─ POST cf-admin /api/chatbot/kb
    │      ↓
    ├─ cf-admin proxies to → POST cf-chatbot /api/admin/kb
    │      Header: X-Admin-Key
    │
    └─ cf-chatbot:
         1. INSERT into D1 kb_entries
            (triggers fire → kb_entries_fts updated automatically)
         2. Generate embedding:
            embedding = await env.AI.run('@cf/baai/bge-small-en-v1.5', { text: content })
         3. Upsert to Vectorize:
            await env.VECTOR_INDEX.upsert([{
              id: entryId,
              values: embedding.data[0],
              metadata: { topic, title, language }
            }])
         4. UPDATE kb_entries SET vector_id = entryId
         5. Return 201 Created
```

### Update Entry

Same as create, but:
1. UPDATE D1 (triggers re-index FTS5)
2. Re-generate embedding (content may have changed)
3. Vectorize upsert with same ID (overwrites existing vector)

### Delete Entry

1. DELETE from D1 (trigger removes from FTS5)
2. `await env.VECTOR_INDEX.deleteByIds([entryId])` — remove from Vectorize

### Toggle Active Status

Setting `is_active = 0` hides the entry from all searches without deleting it:
- D1 queries filter `WHERE is_active = 1`
- FTS5 queries also filter via JOIN with `kb_entries`
- Vectorize vectors remain but are filtered out in post-processing

---

## D1 / Vectorize Consistency Strategy

These two systems can get out of sync if a partial write fails (e.g., D1 write succeeds but Vectorize upsert fails).

### Detection

The `vector_id` column in `kb_entries` tracks whether a Vectorize vector exists for each entry. A `NULL` value means the embedding was never written or was lost.

```sql
-- Find entries without a Vectorize vector
SELECT id, title FROM kb_entries WHERE vector_id IS NULL AND is_active = 1;
```

### Recovery

A re-indexing operation (triggered manually from cf-admin) rebuilds the Vectorize index from D1:

```
POST cf-chatbot /api/admin/kb/reindex
  (Admin role required)

1. Fetch all active kb_entries from D1
2. For each entry without a vector_id:
   a. Generate embedding via Workers AI
   b. Upsert to Vectorize
   c. UPDATE kb_entries SET vector_id = entryId
3. Return summary: { reindexed: N, skipped: M }
```

**When to reindex**:
- After a failed bulk import
- After Vectorize index is manually deleted/recreated
- If chatbot seems to be missing known answers (semantic search returning wrong results)

---

## Content Guidelines

Good KB entries improve retrieval quality:

| Guideline | Why |
|-----------|-----|
| One topic per entry (short, focused) | Better embedding specificity |
| Write in the same language as queries | Spanish entries for Spanish queries |
| Include common synonyms and phrasings | Captures user vocabulary variety |
| Avoid table/markdown formatting | Plain prose embeds better |
| Include specific numbers (prices, times) | FTS5 finds exact numbers reliably |
| Keep entries under 500 words | Long entries dilute embedding quality |

**Minimum KB for good chatbot performance**: 30–50 entries covering all topics, pricing, and policies. Below this, the bot frequently escalates or gives generic answers.
