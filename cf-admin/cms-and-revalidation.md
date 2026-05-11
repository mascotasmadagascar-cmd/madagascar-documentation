# cf-admin — CMS & ISR Revalidation

Content block system, image management, ISR revalidation flow, and the CMS UI components.

---

## Overview

cf-admin is the write endpoint for the public-facing cf-astro website. When hotel staff update content — gallery images, service descriptions, reviews — cf-admin propagates those changes to the live site without a rebuild or redeployment.

The mechanism uses **ISR (Incremental Static Regeneration) revalidation** via the `ASTRO_SERVICE` service binding: cf-admin calls cf-astro internally (zero-latency) to invalidate cached page HTML, so the next visitor gets fresh content.

---

## Content Model

### What Lives Where

| Content Type | Primary Store | Purpose |
|-------------|--------------|---------|
| Hero text, service descriptions, pricing, FAQs | D1 `cms_content` | Structured, versioned, queryable |
| Gallery images, hero images | R2 `madagascar-images` | Binary blobs served via CDN |
| Feature flags | D1 `admin_feature_flags` | Enable/disable site functionality |
| Booking source of truth | Supabase (bookings table) | Relational, shared with public booking flow |

### D1 `cms_content` Table

```sql
CREATE TABLE cms_content (
  id TEXT PRIMARY KEY,       -- Composite key: {page}_{block_id}, e.g., "hero_title_es"
  page TEXT NOT NULL,
  type TEXT,                 -- text | image_url | json
  content TEXT,
  last_updated_by TEXT,      -- Admin email who made the change
  updated_at TEXT
);
```

**CMS key allowlist** (validated before write to prevent arbitrary key injection):

```
hero
services
pricing
gallery
testimonials
faqs
faq_draft
about
contact
franchise
blog_index
seo_*         (prefix pattern)
blog_draft_*  (prefix pattern)
```

Only keys matching the allowlist are accepted. All others are rejected with a 400 error.

---

## ISR Revalidation Flow (`src/lib/cms.ts`, 302 lines)

```
1. Admin edits content block in UI
      ↓
2. API endpoint (POST /api/content/blocks, POST /api/content/services, etc.)
   upserts row in D1 cms_content
      ↓
3. cms.ts: revalidateAstro(env, paths, cmsData) called
      ↓
4. ASTRO_SERVICE.fetch() — zero-latency service binding call to cf-astro
   Headers: Authorization: Bearer {REVALIDATION_SECRET}
   Body: { paths: [...], cmsData: { key: value } }
      ↓ (retry on failure)
   Attempt 1 — immediate
   Attempt 2 — after 300ms
   Attempt 3 — after 600ms
   Attempt 4 — after 900ms
      ↓
5. cf-astro: verifies REVALIDATION_SECRET, deletes ISR KV cache keys for the paths
   Optionally: writes fresh CMS JSON directly into cf-astro KV to bypass D1 lag
      ↓
6. Returns: { status, pathsPurged, cmsKeysWritten }
      ↓
7. Next visitor to cf-astro gets a fresh SSR render (reads updated D1/KV content)
8. cf-astro caches the new HTML in KV — all subsequent visitors get the cached version
```

**Retry policy:** 3 retry attempts with linear backoff: 300ms → 600ms → 900ms delays. Each attempt uses the same ASTRO_SERVICE binding call.

**Safety guard:** `revalidateAstro()` detects if `PUBLIC_ASTRO_URL == SITE_URL` (self-reference) and fails fast with an error rather than calling itself in a loop.

**Content injection:** When `cmsData` is provided, cf-astro optionally writes the fresh CMS JSON into its own KV store, bypassing D1 eventual-consistency lag. This ensures visitors see the new content within milliseconds of the admin saving.

---

## CMS API Endpoints

| Method | Path | Min Role | Purpose |
|--------|------|----------|---------|
| POST | `/api/content/blocks` | admin | Generic CMS block upsert (any allowed key) |
| POST | `/api/content/services` | admin | Update services blocks (specialized handler) |
| POST | `/api/content/reviews` | admin | Update reviews blocks (specialized handler) |
| POST | `/api/media/revalidate` | admin | Trigger ISR revalidation without content change |

### `POST /api/content/blocks` — Request Body

```json
{
  "id": "hero_title_es",
  "page": "hero",
  "type": "text",
  "content": "Bienvenidos a Madagascar Pet Hotel"
}
```

The `id` must match the CMS key allowlist or the request is rejected.

---

## Gallery and Image Management

### Upload Flow

```
1. Admin drags image into GalleryManager.tsx or clicks upload
2. Browser sends: POST /api/media/upload (multipart/form-data, field: file)
3. cf-admin validates: JPEG / PNG / WebP only, max 5MB
4. cf-admin generates a UUID filename: {uuid}.{ext}
5. env.IMAGES.put(filename, imageBuffer, { httpMetadata: { contentType } })
6. D1 cms_content upserted with CDN URL
7. Response returns: https://cdn.madagascarhotelags.com/{uuid}.jpg
8. ISR revalidation triggered for gallery-affected pages
```

**CDN domain:** `https://cdn.madagascarhotelags.com` — custom domain mapped to the `madagascar-images` R2 bucket. R2 egress via CDN custom domain is $0.

**Access control:** The R2 bucket allows public read via the CDN domain. Write access requires the `IMAGES` R2 binding (Workers only — no public upload endpoint).

### Gallery API

| Method | Path | Min Role | Notes |
|--------|------|----------|-------|
| POST | `/api/media/upload` | admin | Upload image to R2 |
| GET | `/api/media/gallery` | admin | Fetch gallery image list (ETag cached) |

`GET /api/media/gallery` uses `withETag()` response helper to support browser-side 304 Not Modified caching.

---

## Feature Flags

Feature flags allow enabling or disabling functionality on cf-astro without a redeployment.

### D1 `admin_feature_flags` Table

```sql
CREATE TABLE admin_feature_flags (
  flag_key TEXT PRIMARY KEY,
  is_enabled INTEGER DEFAULT 0,
  last_updated_by TEXT,      -- Email of admin who last toggled
  created_at TEXT,
  updated_at TEXT
);
```

### Feature Flag Toggle Flow

```
1. Admin toggles flag in cf-admin Settings → Features (FeatureToggles.tsx)
2. POST /api/features/toggle { flag_key, is_enabled }
3. UPDATE D1 admin_feature_flags SET is_enabled = $value
4. Trigger ISR revalidation on cf-astro
5. cf-astro clears its cache for the affected page
6. Next cf-astro request: reads fresh flag value from D1
```

**DEV role required** for `/api/features/toggle`.

---

## CMS UI Components

### Content Studio (`/dashboard/content/`)

| Component | Page | Manages |
|-----------|------|---------|
| `ServicesManager.tsx` | `/dashboard/content/services` | Service names, descriptions, pricing |
| `ReviewsManager.tsx` | `/dashboard/content/reviews` | Customer testimonials — add, edit, toggle visibility |
| `GalleryManager.tsx` | `/dashboard/content/gallery` | Pet photos — drag-drop upload, reorder, delete |
| `ContentTabs.astro` | `/dashboard/content/` | Tab navigation between content sections |

### Gallery Manager (`GalleryManager.tsx`)

- Drag-and-drop file upload
- JPEG / PNG / WebP validation (client-side preview before upload)
- Image reordering (updates `sort_order` in D1)
- Delete (removes R2 key + D1 row + triggers revalidation)
- Shows CDN URL for each image

### Services Manager (`ServicesManager.tsx`)

- Edit service name, description, and price per locale (es / en)
- Saves to D1 `cms_content` with locale-specific keys (e.g., `services_grooming_es`)
- Triggers ISR revalidation for services pages on save

### Reviews Manager (`ReviewsManager.tsx`)

- Add and edit customer testimonial text
- Toggle review visibility (`is_active` field)
- Triggers ISR revalidation for home page testimonial section

---

## Revalidation and Content Propagation Times

| Admin Action | Pages Revalidated | Propagation Time |
|-------------|------------------|-----------------|
| Edit hero text | `/`, `/es/`, `/en/` | <5 seconds |
| Edit service pricing | `/`, `/es/`, `/en/`, `/es/services`, `/en/services` | <5 seconds |
| Upload gallery image | `/`, `/es/`, `/en/` | <5 seconds |
| Edit review | `/`, `/es/`, `/en/` | <5 seconds |
| Toggle feature flag | Affected pages only | <60 seconds (ISR) |

After revalidation, the first visitor to each invalidated page triggers a fresh SSR render. All subsequent visitors receive the newly cached HTML.
