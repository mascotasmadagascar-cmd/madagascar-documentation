# cf-astro — SEO & Internationalization

> Sitemap generation, robots.txt, locale routing, and meta tags.

---

## Locale Architecture

### URL Structure

| Locale | URL Pattern | Default |
|--------|------------|---------|
| Spanish (es) | `/`, `/servicios`, `/reservar` | ✅ Yes |
| English (en) | `/en`, `/en/services`, `/en/booking` | No |

Spanish is the primary locale (root URLs). English pages are prefixed with `/en/`.

### Routing

Astro's file-based routing with locale directories:
```
src/pages/
├── index.astro          # / (Spanish homepage)
├── servicios.astro      # /servicios
├── reservar.astro       # /reservar
├── contacto.astro       # /contacto
├── privacidad.astro     # /privacidad
├── arco.astro           # /arco
├── en/
│   ├── index.astro      # /en (English homepage)
│   ├── services.astro   # /en/services
│   ├── booking.astro    # /en/booking
│   ├── contact.astro    # /en/contact
│   ├── privacy.astro    # /en/privacy
│   └── arco.astro       # /en/arco
```

### Language Detection
- URL-based routing (not browser detection)
- Language switcher in header toggles between `/path` ↔ `/en/path`
- User preference not persisted (stateless, privacy-first)

### `hreflang` Tags
Every page includes alternate language links:
```html
<link rel="alternate" hreflang="es" href="https://madagascarhotelags.com/servicios" />
<link rel="alternate" hreflang="en" href="https://madagascarhotelags.com/en/services" />
<link rel="alternate" hreflang="x-default" href="https://madagascarhotelags.com/servicios" />
```

---

## SEO Configuration

### Meta Tags (Per Page)

Every page includes:
```html
<title>{pageTitle} | Madagascar Pet Hotel</title>
<meta name="description" content="{pageDescription}" />
<meta name="robots" content="index, follow" />
<link rel="canonical" href="{canonicalUrl}" />

<!-- Open Graph -->
<meta property="og:title" content="{pageTitle}" />
<meta property="og:description" content="{pageDescription}" />
<meta property="og:type" content="website" />
<meta property="og:url" content="{canonicalUrl}" />
<meta property="og:image" content="{ogImage}" />
<meta property="og:locale" content="es_MX" />
<meta property="og:locale:alternate" content="en_US" />

<!-- Twitter Card -->
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:title" content="{pageTitle}" />
<meta name="twitter:description" content="{pageDescription}" />
```

### Structured Data (JSON-LD)

```json
{
  "@context": "https://schema.org",
  "@type": "PetStore",
  "name": "Madagascar Pet Hotel",
  "url": "https://madagascarhotelags.com",
  "address": {
    "@type": "PostalAddress",
    "addressLocality": "Aguascalientes",
    "addressCountry": "MX"
  },
  "priceRange": "$$",
  "serviceType": ["Pet Boarding", "Dog Hotel", "Cat Hotel"]
}
```

### Sitemap

Auto-generated `sitemap.xml` listing all public pages in both locales with proper `lastmod` dates.

### robots.txt
```
User-agent: *
Allow: /
Sitemap: https://madagascarhotelags.com/sitemap.xml
```

---

## Performance SEO

| Metric | Target | Strategy |
|--------|--------|---------|
| LCP (Largest Contentful Paint) | <2.5s | ISR cache, edge-served |
| FID (First Input Delay) | <100ms | Minimal JS, Preact islands |
| CLS (Cumulative Layout Shift) | <0.1 | Fixed image dimensions |
| TTFB (Time to First Byte) | <50ms | ISR HIT <10ms, edge compute |

### Astro's Zero-JS Advantage
By default, Astro ships **zero JavaScript** to the browser. Only interactive islands (booking form, carousel, chatbot widget) load JS. This results in:
- Smaller page weight (50–100KB typical)
- Faster FCP/LCP
- Better Core Web Vitals scores
