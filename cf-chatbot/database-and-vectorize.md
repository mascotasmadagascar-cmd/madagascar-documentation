# cf-chatbot — Knowledge Base (D1 & Vectorize)

> Database schema for the RAG pipeline and Vectorize index management.

---

## D1 Schema (chatbot-kb)

The chatbot manages its own isolated D1 database to ensure sub-millisecond read latency for the RAG pipeline.

### Core Tables

#### `kb_entries`
The source of truth for all knowledge base documents.

```sql
CREATE TABLE kb_entries (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,
  category TEXT NOT NULL, -- e.g., 'policies', 'pricing', 'health'
  content TEXT NOT NULL,
  is_active INTEGER DEFAULT 1,
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now'))
);
```

#### `kb_entries_fts`
A Full-Text Search (FTS5) virtual table used for the keyword matching side of the hybrid search.

```sql
CREATE VIRTUAL TABLE kb_entries_fts USING fts5(
  title, 
  content, 
  category UNINDEXED, 
  content='kb_entries', 
  content_rowid='id'
);
```

**Triggers:**
To keep the FTS table perfectly synced with the main table, SQLite triggers automatically manage insertions, updates, and deletions.

```sql
CREATE TRIGGER kb_entries_ai AFTER INSERT ON kb_entries BEGIN
  INSERT INTO kb_entries_fts(rowid, title, content, category) 
  VALUES (new.id, new.title, new.content, new.category);
END;

CREATE TRIGGER kb_entries_ad AFTER DELETE ON kb_entries BEGIN
  INSERT INTO kb_entries_fts(kb_entries_fts, rowid, title, content, category) 
  VALUES('delete', old.id, old.title, old.content, old.category);
END;

CREATE TRIGGER kb_entries_au AFTER UPDATE ON kb_entries BEGIN
  INSERT INTO kb_entries_fts(kb_entries_fts, rowid, title, content, category) 
  VALUES('delete', old.id, old.title, old.content, old.category);
  INSERT INTO kb_entries_fts(rowid, title, content, category) 
  VALUES (new.id, new.title, new.content, new.category);
END;
```

#### `bot_config`
Dynamic configuration settings adjustable via `cf-admin` without needing to redeploy the worker.

```sql
CREATE TABLE bot_config (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  description TEXT,
  updated_at TEXT DEFAULT (datetime('now'))
);
```
Example keys: `system_prompt_override`, `max_context_turns`, `temperature`.

---

## Vectorize Setup (Semantic Search)

Cloudflare Vectorize stores the high-dimensional mathematical representations (embeddings) of the KB entries.

### Index Configuration

```bash
wrangler vectorize create chatbot-kb --dimensions=384 --metric=cosine
```
- **Dimensions**: 384 matches the output of the `@cf/baai/bge-small-en-v1.5` Workers AI model.
- **Metric**: Cosine similarity is the standard for measuring the angle distance between textual embeddings.

### Embedding Generation Workflow

When an admin creates a new KB entry via the `cf-admin` dashboard, the following sequence occurs inside `cf-chatbot`:

1. **Insert D1**: The text is saved to `kb_entries`. (D1 ID = 42)
2. **Generate Embeddings**: 
   ```typescript
   const { data } = await env.AI.run('@cf/baai/bge-small-en-v1.5', {
     text: [kbEntry.content]
   });
   const vector = data[0];
   ```
3. **Insert Vectorize**:
   ```typescript
   await env.VECTORIZE.insert([{
     id: `kb_42`, // Link back to D1 ID
     values: vector,
     metadata: { category: kbEntry.category }
   }]);
   ```

### Vector Search Query

During a user chat, the retrieval step looks like this:

```typescript
// 1. Embed user query
const queryVector = await getEmbedding(userQuery);

// 2. Search Vectorize
const vectorMatches = await env.VECTORIZE.query(queryVector, { 
  topK: 5,
  returnMetadata: true 
});

// 3. Extract D1 IDs from results (e.g., "kb_42" -> 42)
const matchedIds = vectorMatches.matches.map(m => parseInt(m.id.split('_')[1]));

// 4. Fetch full text from D1
const docs = await env.D1.prepare(
  `SELECT * FROM kb_entries WHERE id IN (${matchedIds.join(',')})`
).all();
```

---

## Data Synchronization & Consistency

Because text data lives in D1 and vector data lives in Vectorize, network errors could theoretically cause them to drift out of sync. 

**Resilience Strategy:**
- All writes are processed sequentially within the `api/admin/kb` endpoints.
- If D1 insertion succeeds but Vectorize insertion fails, the API returns a 500 error. The admin dashboard UI indicates the failure, prompting a manual retry.
- A future enhancement could utilize Cloudflare Queues to guarantee eventual consistency for embedding generation.
