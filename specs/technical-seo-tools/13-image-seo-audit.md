# Tool 13: Image SEO Audit

## What This Replicates
- Screaming Frog image analysis
- ahrefs image SEO checker
- GTmetrix image optimization report

## What It Does
Audits all images on a page for SEO best practices: alt text, file sizes, modern formats (WebP/AVIF), lazy loading, descriptive filenames, and dimension attributes. AEO angle: AI engines use alt text and surrounding context to understand images; proper image SEO improves content comprehension.

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-image-audit
```

### Request
```json
{
  "url": "https://example.com/blog/crm-guide"
}
```

### Response
```json
{
  "score": 62,
  "summary": {
    "totalImages": 14,
    "withAlt": 9,
    "withoutAlt": 5,
    "modernFormat": 8,
    "legacyFormat": 6,
    "lazyLoaded": 10,
    "oversized": 3,
    "totalImageSize": "4.2MB",
    "avgImageSize": "307KB"
  },
  "images": [
    {
      "src": "https://example.com/images/crm-dashboard.webp",
      "alt": "CRM dashboard showing sales pipeline and revenue metrics",
      "width": 1200,
      "height": 630,
      "fileSize": "85KB",
      "format": "webp",
      "loading": "lazy",
      "inViewport": false,
      "issues": [],
      "aeoScore": "good"
    },
    {
      "src": "https://example.com/images/IMG_4532.jpg",
      "alt": "",
      "width": null,
      "height": null,
      "fileSize": "1.8MB",
      "format": "jpeg",
      "loading": "eager",
      "inViewport": false,
      "issues": [
        { "code": "IMG_NO_ALT", "severity": "warning", "title": "Missing alt text" },
        { "code": "IMG_OVERSIZED", "severity": "warning", "title": "Image is 1.8MB (recommend < 200KB)" },
        { "code": "IMG_NO_DIMENSIONS", "severity": "warning", "title": "Missing width/height attributes (causes CLS)" },
        { "code": "IMG_LEGACY_FORMAT", "severity": "info", "title": "JPEG could be converted to WebP for 25-35% savings" },
        { "code": "IMG_GENERIC_FILENAME", "severity": "info", "title": "Filename 'IMG_4532.jpg' is not descriptive" },
        { "code": "IMG_NOT_LAZY", "severity": "info", "title": "Below-fold image is not lazy-loaded" }
      ]
    }
  ],
  "issues": [ ... ]
}
```

---

## Validation Rules

| Code | Severity | Check |
|------|----------|-------|
| `IMG_NO_ALT` | warning | Missing alt attribute |
| `IMG_EMPTY_ALT` | info | Alt is empty string (OK for decorative, not for content images) |
| `IMG_ALT_TOO_LONG` | info | Alt text exceeds 125 characters |
| `IMG_ALT_KEYWORD_STUFFED` | warning | Alt text appears keyword-stuffed |
| `IMG_OVERSIZED` | warning | File size exceeds 200KB |
| `IMG_VERY_OVERSIZED` | critical | File size exceeds 1MB |
| `IMG_NO_DIMENSIONS` | warning | Missing width/height (causes CLS) |
| `IMG_LEGACY_FORMAT` | info | Using JPEG/PNG instead of WebP/AVIF |
| `IMG_NOT_LAZY` | info | Below-fold image without loading="lazy" |
| `IMG_LCP_LAZY` | warning | Above-fold (LCP candidate) image has loading="lazy" (hurts LCP) |
| `IMG_GENERIC_FILENAME` | info | Filename like IMG_1234.jpg, screenshot.png, image1.jpg |
| `IMG_BROKEN` | critical | Image returns 404 or error |
| `IMG_NO_SRCSET` | info | No srcset for responsive images |
| `AEO_ALT_DESCRIPTIVE` | passed | Alt text is descriptive and contextual |
| `AEO_ALT_MISSING_CONTEXT` | warning | Alt text doesn't describe what AI needs to understand the content |

---

## Scoring

```
Score starts at 100:
- Each image without alt: -5 (max -25)
- Each oversized image: -3 (max -15)
- Each missing dimension: -2 (max -10)
- Each broken image: -10 (max -20)
- No modern formats used at all: -10
- LCP image is lazy loaded: -10
- More than 50% generic filenames: -5
```
