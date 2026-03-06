# Tool 05: HTTP Header Inspector

## What This Replicates
- SecurityHeaders.com
- RedBot.org header checker
- WebPageTest response header analysis

## What It Does
Makes an HTTP request and analyzes all response headers. Checks for SEO-critical headers (X-Robots-Tag, Link, Cache-Control), security headers (CSP, HSTS, X-Frame-Options), and performance headers (compression, caching). AEO angle: proper headers ensure AI crawlers can efficiently access and cache your content.

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-header-inspector
```

### Request
```json
{
  "url": "https://example.com/products/widget",
  "method": "GET",
  "followRedirects": false
}
```

### Response
```json
{
  "score": 88,
  "request": {
    "method": "GET",
    "url": "https://example.com/products/widget",
    "finalUrl": "https://example.com/products/widget",
    "statusCode": 200,
    "statusText": "OK",
    "protocol": "h2",
    "timing": { "dns": 12, "connect": 45, "ttfb": 180, "total": 220 }
  },
  "headers": {
    "raw": {
      "content-type": "text/html; charset=utf-8",
      "content-encoding": "br",
      "cache-control": "public, max-age=3600, s-maxage=86400",
      "strict-transport-security": "max-age=63072000; includeSubDomains; preload",
      "x-content-type-options": "nosniff",
      "x-frame-options": "SAMEORIGIN",
      "content-security-policy": "default-src 'self'; script-src 'self' 'unsafe-inline' https://cdn.example.com",
      "referrer-policy": "strict-origin-when-cross-origin",
      "permissions-policy": "camera=(), microphone=(), geolocation=()",
      "x-robots-tag": "index, follow",
      "link": "<https://example.com/products/widget>; rel=\"canonical\"",
      "vary": "Accept-Encoding",
      "server": "nginx/1.25",
      "x-powered-by": "Next.js"
    },
    "grouped": {
      "seo": {
        "x-robots-tag": { "value": "index, follow", "status": "good" },
        "link-canonical": { "value": "https://example.com/products/widget", "status": "good" },
        "content-type": { "value": "text/html; charset=utf-8", "status": "good" }
      },
      "security": {
        "strict-transport-security": { "value": "max-age=63072000...", "status": "good", "grade": "A" },
        "content-security-policy": { "value": "default-src 'self'...", "status": "good", "grade": "B" },
        "x-content-type-options": { "value": "nosniff", "status": "good" },
        "x-frame-options": { "value": "SAMEORIGIN", "status": "good" },
        "referrer-policy": { "value": "strict-origin-when-cross-origin", "status": "good" },
        "permissions-policy": { "value": "camera=()...", "status": "good" }
      },
      "performance": {
        "content-encoding": { "value": "br", "status": "good", "note": "Brotli compression enabled" },
        "cache-control": { "value": "public, max-age=3600...", "status": "good" },
        "vary": { "value": "Accept-Encoding", "status": "good" }
      },
      "info": {
        "server": { "value": "nginx/1.25", "status": "warning", "note": "Server version exposed" },
        "x-powered-by": { "value": "Next.js", "status": "warning", "note": "Technology stack exposed" }
      }
    }
  },
  "issues": [ ... ]
}
```

---

## Validation Rules

### SEO Headers
| Code | Severity | Check |
|------|----------|-------|
| `XROBOTS_NOINDEX` | critical | X-Robots-Tag contains noindex |
| `XROBOTS_NOSNIPPET` | warning | X-Robots-Tag contains nosnippet (blocks AI excerpts) |
| `XROBOTS_UNAVAILABLE_AFTER` | info | X-Robots-Tag has unavailable_after date |
| `LINK_CANONICAL_PRESENT` | passed | Canonical URL in Link header |
| `LINK_CANONICAL_MISMATCH` | warning | Header canonical differs from HTML canonical |
| `CONTENT_TYPE_MISSING` | critical | No Content-Type header |
| `CONTENT_TYPE_NO_CHARSET` | warning | Content-Type missing charset |
| `HREFLANG_IN_HEADER` | info | Hreflang alternate links in headers |

### Security Headers
| Code | Severity | Check |
|------|----------|-------|
| `HSTS_MISSING` | warning | No Strict-Transport-Security header |
| `HSTS_SHORT_MAXAGE` | warning | HSTS max-age less than 1 year |
| `HSTS_GOOD` | passed | HSTS properly configured |
| `CSP_MISSING` | warning | No Content-Security-Policy |
| `CSP_UNSAFE_INLINE` | info | CSP allows unsafe-inline (common but not ideal) |
| `XCTO_MISSING` | warning | No X-Content-Type-Options |
| `XFO_MISSING` | info | No X-Frame-Options |
| `RP_MISSING` | info | No Referrer-Policy |
| `PP_MISSING` | info | No Permissions-Policy |
| `SERVER_VERSION_EXPOSED` | info | Server header reveals version number |
| `POWERED_BY_EXPOSED` | info | X-Powered-By reveals technology |

### Performance Headers
| Code | Severity | Check |
|------|----------|-------|
| `NO_COMPRESSION` | warning | No content-encoding (gzip/br) |
| `GZIP_ONLY` | info | Using gzip but not Brotli (br is 15-25% smaller) |
| `BROTLI_ENABLED` | passed | Brotli compression active |
| `NO_CACHE_CONTROL` | warning | No Cache-Control header |
| `NO_CACHE_SET` | info | Cache-Control: no-cache or no-store on HTML |
| `GOOD_CACHE_POLICY` | passed | Appropriate caching configured |
| `MISSING_VARY` | info | No Vary header (may cause cache issues) |

### AEO Headers
| Code | Severity | Check |
|------|----------|-------|
| `AEO_NOSNIPPET_HEADER` | critical | X-Robots-Tag nosnippet blocks AI from excerpting |
| `AEO_MAX_SNIPPET_RESTRICTED` | warning | max-snippet is set too low for meaningful AI extraction |
| `AEO_GOOD_CACHING` | passed | Good cache policy helps AI crawlers be efficient |
| `AEO_NO_COMPRESSION_SLOW` | warning | No compression makes pages slow to crawl (AI bots have time limits) |

---

## Scoring

```
Score starts at 100:
- X-Robots-Tag noindex: -25
- Missing HSTS: -5
- Missing CSP: -5
- Missing X-Content-Type-Options: -3
- No compression: -10
- No Cache-Control: -5
- Server/Powered-By exposed: -2 each
- nosnippet blocking AI: -15
- Missing Content-Type charset: -3
```
