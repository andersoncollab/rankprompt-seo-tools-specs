# Tool 17: Hreflang Validator

## What This Replicates
- Aleyda Solis hreflang generator
- Merkle hreflang tag testing tool
- TechnicalSEO.com hreflang checker

## What It Does
Validates hreflang implementation across HTML tags, HTTP headers, and XML sitemaps. Checks for return tag compliance, valid language/region codes, and self-referencing tags. Critical for multilingual/multi-region sites.

---

## Backend API

### Endpoint
```
POST /functions/v1/seo-tool-hreflang-validator
```

### Request
```json
{
  "url": "https://example.com/products/widget",
  "checkReturnTags": true
}
```

### Response
```json
{
  "score": 60,
  "implementations": {
    "htmlTags": [
      { "hreflang": "en", "href": "https://example.com/products/widget" },
      { "hreflang": "es", "href": "https://example.com/es/productos/widget" },
      { "hreflang": "fr", "href": "https://example.com/fr/produits/widget" },
      { "hreflang": "x-default", "href": "https://example.com/products/widget" }
    ],
    "httpHeaders": [],
    "sitemapEntries": []
  },
  "returnTagCheck": [
    {
      "hreflang": "es",
      "href": "https://example.com/es/productos/widget",
      "returnTagPresent": true,
      "returnTagCorrect": true,
      "pointsBackTo": "https://example.com/products/widget"
    },
    {
      "hreflang": "fr",
      "href": "https://example.com/fr/produits/widget",
      "returnTagPresent": false,
      "returnTagCorrect": false,
      "issue": "French page does not have a return hreflang tag pointing back to the English page"
    }
  ],
  "issues": [
    {
      "code": "HREFLANG_NO_RETURN_TAG",
      "severity": "critical",
      "title": "French page missing return hreflang tag",
      "message": "https://example.com/fr/produits/widget does not contain a hreflang tag pointing back to the English version. Google requires bidirectional hreflang tags."
    },
    {
      "code": "HREFLANG_NO_SELF_REFERENCE",
      "severity": "info",
      "title": "Consider adding self-referencing hreflang tags on alternate pages"
    }
  ]
}
```

---

## Validation Rules

| Code | Severity | Check |
|------|----------|-------|
| `HREFLANG_NONE_FOUND` | info | No hreflang tags found (only an issue for multilingual sites) |
| `HREFLANG_INVALID_CODE` | critical | Language code not valid ISO 639-1 |
| `HREFLANG_INVALID_REGION` | warning | Region code not valid ISO 3166-1 Alpha 2 |
| `HREFLANG_NO_RETURN_TAG` | critical | Target page doesn't have a return hreflang tag |
| `HREFLANG_RETURN_MISMATCH` | critical | Return tag points to a different URL |
| `HREFLANG_NO_SELF` | warning | Missing self-referencing hreflang tag |
| `HREFLANG_NO_X_DEFAULT` | warning | No x-default fallback specified |
| `HREFLANG_DUPLICATE_LANG` | warning | Same language code appears multiple times |
| `HREFLANG_RELATIVE_URL` | warning | Using relative URLs (should be absolute) |
| `HREFLANG_MIXED_IMPLEMENTATION` | info | Using multiple implementation methods (HTML + headers + sitemap) |
| `HREFLANG_TARGET_404` | critical | Hreflang target URL returns 404 |
| `HREFLANG_TARGET_NOINDEX` | warning | Hreflang target has noindex |
| `AEO_HREFLANG_HELPS_AI` | passed | Proper hreflang helps AI engines serve the right language version |

---

## Scoring

```
Score starts at 100 (only scored if hreflang tags are present):
- Invalid language code: -20 each
- Missing return tag: -15 each
- No x-default: -10
- No self-referencing: -5
- Target returns 404: -15
- Target has noindex: -10
- Relative URLs: -5
- If no hreflang at all: score = N/A (not applicable)
```
