# cf-astro — Bindings & Secrets

> Complete wrangler.toml documentation with all bindings and secrets.

---

## Wrangler Configuration

### Core Settings
```toml
name = "cf-astro"
compatibility_date = "2025-04-01"
compatibility_flags = ["nodejs_compat"]
pages_build_output_dir = "./dist"
```

### Workers AI
```toml
[ai]
binding = "AI"
```
Used for: Embedding generation, content summarization (if enabled).

---

## Bindings

### D1 Databases

| Binding | Database | Purpose |
|---------|----------|---------|
| `DB` | `madagascar-db` | Feature flags, site metadata, CMS content |

```toml
[[d1_databases]]
binding = "DB"
database_name = "madagascar-db"
database_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### KV Namespaces

| Binding | Namespace | Purpose |
|---------|-----------|---------|
| `ISR_CACHE` | `MADAGASCAR_ISR_CACHE` | Rendered HTML cache (ISR) |

```toml
[[kv_namespaces]]
binding = "ISR_CACHE"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

### R2 Buckets

| Binding | Bucket | Access | Purpose |
|---------|--------|--------|---------|
| `IMAGES` | `madagascar-images` | Public via Worker | CMS images, pet photos |
| `ARCO_DOCS` | `arco-documents` | Private | Legal/identity documents |

```toml
[[r2_buckets]]
binding = "IMAGES"
bucket_name = "madagascar-images"

[[r2_buckets]]
binding = "ARCO_DOCS"
bucket_name = "arco-documents"
```

### Queue Producers

| Binding | Queue | Purpose |
|---------|-------|---------|
| `EMAIL_QUEUE` | `madagascar-emails` | Async email dispatch |

```toml
[[queues.producers]]
binding = "EMAIL_QUEUE"
queue = "madagascar-emails"
```

### Analytics Engine

| Binding | Dataset | Purpose |
|---------|---------|---------|
| `ANALYTICS` | `madagascar_analytics` | Edge telemetry (page views, conversions) |

```toml
[[analytics_engine_datasets]]
binding = "ANALYTICS"
dataset = "madagascar_analytics"
```

---

## Environment Variables

### Public (Exposed to Client)

| Variable | Description |
|----------|-------------|
| `PUBLIC_SENTRY_DSN` | Sentry DSN for client-side error tracking |
| `PUBLIC_TURNSTILE_SITE_KEY` | Cloudflare Turnstile site key |
| `PUBLIC_SITE_URL` | Canonical site URL |

### Server-Only Secrets

| Secret | Description | Rotation |
|--------|-------------|----------|
| `SUPABASE_URL` | Supabase project URL | Never |
| `SUPABASE_SERVICE_KEY` | Supabase service role key (bypasses RLS) | On demand |
| `DATABASE_URL` | PostgreSQL connection string | On demand |
| `BETTERSTACK_SOURCE_TOKEN` | BetterStack/Logtail source token | On demand |
| `TURNSTILE_SECRET_KEY` | Cloudflare Turnstile server validation key | On demand |
| `UPSTASH_REDIS_REST_URL` | Upstash Redis REST endpoint | Never |
| `UPSTASH_REDIS_REST_TOKEN` | Upstash Redis auth token | On demand |
| `REVALIDATION_SECRET` | Shared secret for ISR webhook auth | On rotation |
| `SENTRY_DSN` | Sentry DSN (server-side) | Never |
| `SENTRY_AUTH_TOKEN` | Sentry auth for source map upload | On demand |

---

## Observability

```toml
[observability]
enabled = true

[observability.logs]
enabled = true
head_sampling_rate = 0.01  # 1% sampling (high traffic)
invocation_logs = true
```

---

## Environment Overrides

```toml
[env.preview]
name = "cf-astro-preview"
# Preview-specific bindings can override defaults

[env.production]
# Production uses the main config
```
