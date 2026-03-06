# Tool 07: Canonical URL Checker

## What This Replicates
- Yoast canonical checker
- Screaming Frog canonical analysis
- ContentKing canonical monitoring

## What It Does
Analyzes canonical URL implementation across a page. Checks for conflicts between HTML canonical, HTTP Link header canonical, and sitemap URLs. Detects common canonical mistakes that cause duplicate content issues and confuse AI engines about which version of a page to cite.

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-canonical-checker
```

### Request
```json
{
  "url": "https://example.com/products/widget?color=blue&utm_source=google"
}
```

### Response
```json
{
  "score": 75,
  "currentUrl": "https://example.com/products/widget?color=blue&utm_source=google",
  "canonicals": {
    "htmlTag": {
      "found": true,
      "value": "https://example.com/products/widget",
      "location": "<link rel='canonical'> in <head>",
      "line": 12
    },
    "httpHeader": {
      "found": true,
      "value": "https://example.com/products/widget",
      "header": "Link: <https://example.com/products/widget>; rel=\"canonical\""
    },
    "sitemapUrl": {
      "found": true,
      "value": "https://example.com/products/widget",
      "sitemapFile": "https://example.com/sitemap-products.xml"
    }
  },
  "analysis": {
    "allMatch": true,
    "resolvedCanonical": "https://example.com/products/widget",
    "selfReferencing": false,
    "pointsToExternalDomain": false,
    "currentUrlMatchesCanonical": false,
    "parameterStripping": {
      "strippedParams": ["color", "utm_source"],
      "note": "Canonical removes query parameters, which is correct for UTM params but may lose 'color' variant info"
    }
  },
  "crossChecks": {
    "canonicalReturns200": true,
    "canonicalIsIndexable": true,
    "canonicalHasSelfCanonical": true,
    "noRedirectLoop": true,
    "canonicalInSitemap": true
  },
  "issues": [
    {
      "code": "CANONICAL_STRIPS_MEANINGFUL_PARAM",
      "severity": "warning",
      "title": "Canonical strips 'color' parameter",
      "message": "The canonical URL removes ?color=blue, which may represent a distinct product variant. If this is a separate page with unique content, it should have its own canonical.",
      "aeoImpact": "AI engines may conflate all color variants as one product, missing specificity in answers."
    }
  ]
}
```

---

## Validation Rules

### Core Canonical Checks
| Code | Severity | Check |
|------|----------|-------|
| `CANONICAL_MISSING` | warning | No canonical tag found (HTML or header) |
| `CANONICAL_EMPTY` | critical | Canonical tag exists but href is empty |
| `CANONICAL_RELATIVE` | warning | Canonical is a relative URL (should be absolute) |
| `CANONICAL_MULTIPLE` | critical | Multiple canonical tags found |
| `CANONICAL_HTML_HEADER_MISMATCH` | critical | HTML and HTTP header canonicals point to different URLs |
| `CANONICAL_SELF_REFERENCING` | passed | Canonical points to itself (best practice) |
| `CANONICAL_NOT_SELF` | info | Canonical points to a different URL |

### Cross-Validation Checks
| Code | Severity | Check |
|------|----------|-------|
| `CANONICAL_RETURNS_ERROR` | critical | Canonical URL returns 4xx or 5xx |
| `CANONICAL_REDIRECTS` | warning | Canonical URL redirects to another URL |
| `CANONICAL_NOT_INDEXABLE` | critical | Canonical URL has noindex |
| `CANONICAL_CHAIN` | critical | Canonical URL points to yet another canonical (chain) |
| `CANONICAL_NOT_IN_SITEMAP` | info | Canonical URL not found in sitemap |
| `CANONICAL_SITEMAP_MISMATCH` | warning | Sitemap lists a different URL than the canonical |
| `CANONICAL_EXTERNAL` | warning | Canonical points to a different domain |
| `CANONICAL_HTTP_VS_HTTPS` | critical | Canonical uses HTTP instead of HTTPS |
| `CANONICAL_WWW_MISMATCH` | warning | Canonical www/non-www doesn't match site preference |

### Parameter-Specific Checks
| Code | Severity | Check |
|------|----------|-------|
| `CANONICAL_STRIPS_UTM` | passed | Correctly strips UTM/tracking parameters |
| `CANONICAL_STRIPS_MEANINGFUL_PARAM` | warning | Strips parameters that may represent unique content |
| `CANONICAL_KEEPS_TRACKING_PARAMS` | warning | Canonical retains UTM or session parameters |
| `CANONICAL_PAGINATION` | info | Page is paginated; check rel=prev/next or canonical strategy |

### AEO-Specific
| Code | Severity | Check |
|------|----------|-------|
| `AEO_CANONICAL_CLARITY` | passed | Clear canonical signals help AI engines identify the primary content version |
| `AEO_CANONICAL_CONFUSION` | warning | Conflicting canonical signals may cause AI to cite the wrong page version |
| `AEO_DUPLICATE_CONTENT_RISK` | warning | Without proper canonicals, AI engines may cite duplicate pages unpredictably |

---

## Frontend Display

### Canonical Map Visualization
```
+----------------------------------------------------+
| Current URL:                                        |
| https://example.com/products/widget?color=blue      |
|                                                     |
| HTML Canonical ----+                                |
|                    +--> https://example.com/         |
| HTTP Header -------+    products/widget              |
|                                                     |
| Sitemap URL -------+--> https://example.com/         |
|                         products/widget   [MATCH]    |
|                                                     |
| Cross-checks:                                       |
| [v] Canonical returns 200                           |
| [v] Canonical is indexable                          |
| [v] Canonical has self-referencing canonical         |
| [v] Canonical is in sitemap                         |
| [!] Strips 'color' parameter                       |
+----------------------------------------------------+
```

---

## Scoring

```
Score starts at 100:
- No canonical at all: -20
- Multiple conflicting canonicals: -25
- HTML/header mismatch: -25
- Canonical returns error: -30
- Canonical redirects: -15
- Canonical has noindex: -30
- Canonical chain: -20
- HTTP instead of HTTPS: -15
- Retains tracking params: -5
- Strips meaningful params: -5
- Not in sitemap: -3
```
