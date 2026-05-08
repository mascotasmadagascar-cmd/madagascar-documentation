# cf-admin — CMS & ISR Revalidation

> Content management architecture, KV injection pipeline, image upload flow, and feature flags.

---

## Overview

cf-admin acts as the **content writer** for the public-facing cf-astro website. When hotel staff update pricing, gallery images, hero text, or feature flags, cf-admin propagates those changes to the live website without a rebuild or redeployment.

The mechanism is a **2-tier KV injection pipeline** that solves a specific problem: D1 has eventual-consistency read replicas, which means a write to D1 might not be immediately visible to the next D1 read. By writing new content to KV simultaneously, cf-astro gets fresh content from KV (strongly consistent) before D1 replication catches up.

---

## Content Model

### What Lives Where

| Content Type | Primary Store | Why |
|-------------|--------------|-----|
| Hero text, pricing, service descriptions | D1 `cms_content` table | Structured, versioned, queryable |
| Gallery images, hero images | R2 `madagascar-images` bucket | Binary blobs, CDN-served |
| Feature flags | D1 `admin_feature_flags` table | Structured with 3-layer cache |
| Testimonials / reviews | Supabase (complex relational) | Requires JOIN with authors |

### D1 `cms_content` Table Structure

```sql
CREATE TABLE cms_content (
  id TEXT PRIMARY KEY,           -- 'hero_title_es', 'service_grooming_price', etc.
  content_type TEXT NOT NULL,    -- 'text' | 'html' | 'json' | 'number'
  value TEXT NOT NULL,           -- The actual content
  locale TEXT,                   -- 'es' | 'en' | NULL (locale-agnostic)
  updated_at INTEGER DEFAULT (unixepoch()),
  updated_by TEXT                -- Admin email who made the change
);
```

---

## The 2-Tier KV Injection Pipeline

When admin saves a content edit, this sequence runs:

```
1. cf-admin: UPDATE D1 cms_content SET value = $newValue WHERE id = $contentId
   → D1 write propagates to read replicas (eventual consistency, ~1-5s)

2. cf-admin: PUT KV ISR_CACHE key = 'cms:{contentId}' value = $newValue TTL = 3600
   → KV write is immediately consistent (strongly consistent)

3. cf-admin: POST https://madagascarhotelags.com/api/revalidate
   → Deletes the ISR page cache keys so next visitor gets fresh HTML
```

**Why both D1 and KV?**
- D1 is the **source of truth** — persistent, structured, survives KV TTL expiry
- KV is the **hot path** — cf-astro reads KV first (1ms), falls back to D1 (5–15ms) if KV misses
- Direct KV injection means content is live in <1 second after admin saves

---

## ISR Revalidation: `revalidateAstro()`

When content changes, the cached HTML pages on cf-astro must be invalidated. cf-admin calls a unified helper:

```typescript
// src/lib/cms.ts
await revalidateAstro(env, ['/es/', '/en/']);
```

This helper:
1. **Auto-expands** base paths to all locale variants (`'/'` → `['/', '/es/', '/en/']`)
2. Fires `POST {PUBLIC_ASTRO_URL}/api/revalidate` with `Authorization: Bearer {REVALIDATION_SECRET}`
3. cf-astro's revalidate endpoint verifies the secret and deletes all matching `isr:{buildId}:{path}` KV keys
4. **Retry**: 3 attempts with exponential backoff (1s → 2s → 4s delays)

### What Triggers Revalidation

| Admin Action | Pages Revalidated |
|-------------|-----------------|
| Edit hero text | `/es/`, `/en/` |
| Edit service pricing | `/es/`, `/en/`, `/es/services`, `/en/services` |
| Upload gallery image | `/es/`, `/en/` |
| Edit review/testimonial | `/es/`, `/en/` |
| Edit FAQ content | `/es/`, `/en/` |

After revalidation:
1. Next visitor hits cf-astro for the invalidated page
2. cf-astro: KV miss → full SSR render (reads fresh D1/KV content)
3. cf-astro: Saves new HTML to KV → all subsequent visitors get cached HTML

**User experience**: Content is visually live within 1–10 seconds of admin saving. No redeploy needed.

---

## Image Upload Flow (R2 CDN)

Gallery images and hero images are stored in Cloudflare R2, served via a custom CDN domain.

```
1. Admin drags image into cf-admin Gallery Manager
2. Browser sends: POST /api/media/upload (multipart/form-data)
3. cf-admin: validate file type (JPEG/PNG/WebP only) + size (<5MB)
4. cf-admin: generate UUID filename: {uuid}.{ext}
5. cf-admin: PUT env.IMAGES.put(filename, imageBuffer, { httpMetadata: { contentType } })
6. cf-admin: returns CDN URL: https://cdn.madagascarhotelags.com/{filename}
7. Admin sees preview with CDN URL
8. Admin saves → cf-admin updates D1 cms_content with new image URL
9. cf-admin: triggers ISR revalidation for affected pages
```

**CDN domain**: `cdn.madagascarhotelags.com` — a custom domain mapped to the R2 bucket. Egress is $0 (R2 advantage over S3).

**Access control**: The R2 `madagascar-images` bucket has public read access via the CDN domain. Write access requires the R2 binding (Workers only — no public upload endpoint).

---

## Feature Flags

Feature flags allow enabling/disabling functionality on cf-astro without a redeployment.

### 3-Layer Cache Architecture

```
Request hits cf-astro → check in-memory cache (10 second TTL, per isolate)
    │
    HIT: return cached value (0 network calls)
    │
    MISS: check Cache API (60 second TTL, shared across isolates on same PoP)
         │
         HIT: return cached value (sub-ms)
         │
         MISS: query D1 admin_feature_flags table
              │
              Return value → write to Cache API → write to in-memory
```

**Why 3 layers?**: Each Cloudflare Worker isolate has its own memory. Cache API is shared within a PoP. D1 is globally authoritative. This hierarchy reduces D1 reads by ~99% under normal load.

### Flag Toggle Flow

```
1. Admin toggles flag in cf-admin Settings → Features
2. cf-admin: POST /api/features/toggle { flagId, enabled }
3. cf-admin: UPDATE D1 admin_feature_flags SET enabled = $value
4. cf-admin: triggers cf-astro ISR revalidation
5. cf-astro: clears its Cache API entries for that flag (via revalidate endpoint)
6. Next cf-astro request: D1 fresh read → new flag value propagated
```

**TTL cascade**: Even without explicit cache clearing, flags propagate within 60 seconds via Cache API TTL + 10 seconds via in-memory TTL.

---

## Content Studio Hub

The cf-admin CMS is organized into a Content Studio at `/dashboard/content/`:

| Section | Path | Manages |
|---------|------|---------|
| Hero | `/dashboard/content/hero` | Hero image, headline, subheadline (es + en) |
| Services | `/dashboard/content/services` | Service names, descriptions, pricing (es + en) |
| Gallery | `/dashboard/content/gallery` | Pet photos — drag-drop upload, ordering, delete |
| Reviews | `/dashboard/content/reviews` | Customer testimonials — add, edit, toggle visibility |
| FAQ | `/dashboard/content/faq` | FAQ questions and answers (es + en) |
| About | `/dashboard/content/about` | About section content |

Each section reads live data from D1/KV on load and writes changes with immediate ISR revalidation.
