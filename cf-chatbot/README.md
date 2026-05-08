# cf-chatbot — AI Chatbot

> Intelligent customer support agent with RAG pipeline and multi-model fallback.

---

## Quick Reference

| Property | Value |
|----------|-------|
| **Framework** | Vanilla TypeScript |
| **Runtime** | Cloudflare Workers (V8 isolate) |
| **Vector DB** | Cloudflare Vectorize |
| **Knowledge Base** | Cloudflare D1 (FTS5) |
| **Primary LLM** | Gemini 2.0 Flash / Claude 3.5 Haiku |
| **Fallback LLM** | Workers AI (`@cf/meta/llama-3-8b-instruct`) |
| **Deploy** | `npm run deploy` |

## Purpose
The AI agent answering customer inquiries via WhatsApp and the website widget:
- 🧠 RAG (Retrieval-Augmented Generation) pipeline combining dense vectors and sparse text search.
- 🔀 Multi-model fallback chain ensures high availability if an AI provider goes down.
- ⚡ Sub-second response times using Upstash Redis semantic caching.
- 🕵️‍♂️ Intent detection to route specific queries (e.g., booking requests) to specialized prompt templates.
- 📊 Comprehensive usage and cost analytics synced asynchronously to Supabase.

## Key Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `@google/genai` | ^0.1.2 | Gemini API client |
| `@anthropic-ai/sdk` | ^0.20.1 | Claude API client |
| `@upstash/redis` | ^1.29.0 | Semantic caching |
| `hono` | ^4.2.0 | Lightweight router |

## File Structure
```
cf-chatbot/
├── src/
│   ├── index.ts              # Entry point, Hono router, middleware
│   ├── core/
│   │   ├── ai.ts             # Multi-model orchestrator (Gemini/Claude/Meta)
│   │   ├── rag.ts            # Hybrid search pipeline (Vectorize + FTS5)
│   │   ├── intent.ts         # Query classification and prompt selection
│   │   └── prompt.ts         # System prompts and templates
│   ├── storage/
│   │   ├── d1.ts             # Knowledge base retrieval
│   │   ├── supabase.ts       # Analytics and conversation history
│   │   └── cache.ts          # Upstash Redis caching layer
│   ├── api/
│   │   ├── webhook.ts        # Client facing API (WhatsApp, Web widget)
│   │   └── admin.ts          # Internal API (Proxy for cf-admin)
│   └── types/                # Shared TypeScript interfaces
├── wrangler.jsonc            # Cloudflare configuration
├── vitest.config.ts          # Test configuration
└── package.json
```
